# Source Adapter Runtime Contract

This document describes source-specific requirements for delivering records to
the durable ingest lane. The adapters and their configuration are not
implemented in this repository. Start with the shared
[`processing-lifecycle.md`](processing-lifecycle.md) contract: acknowledge a
source position only after the ingest event is durably published, unless that
source cannot support deferred acknowledgement and its weaker guarantee is
explicitly accepted.

## Common adapter contract

Every source adapter must:

1. Expose a stable source identity and report source position separately from
   the event payload.
2. Produce the same `event_id` and `partition_key` when the same source record
   is retried. Prefer a source-provided immutable record ID; otherwise derive
   identity from source identity and stable source position, not arrival time.
3. Apply bounded fetch, in-flight, and local buffering limits. Define what
   happens when the durable ingest lane is unavailable or the buffer is full.
4. Publish before advancing a cursor, committing an offset, acknowledging a
   delivery, or marking a file complete.
5. Preserve enough source metadata to diagnose and replay a record: source
   identity, partition or file identity, source position, and adapter version.
6. Expose lag/age, fetch and publish failures, retries, duplicate estimates,
   cursor/offset progress, and time spent backpressured without logging
   credentials or unrestricted event payloads.
7. Stop or fence source ownership during drain/rebalance. A former owner must
   not continue committing after a replacement takes the lease.

`TBD` values in the example are deployment inputs. They must be set and
validated against source quotas, ingest throughput, outage tolerance, and
recovery objectives before production.

## HTTP polling API

- Require an API contract for pagination and resumability: opaque cursor,
  monotonically increasing sequence, or a documented time-window overlap.
  Offset pagination over a changing dataset is not a safe recovery cursor.
- Persist the cursor only after every returned record has a durable ingest
  publish confirmation. If a page is only partially published, replay the
  page; record-level IDs make this safe under at-least-once delivery.
- For time-window polling, define overlap, late-arrival horizon, stable
  deduplication key, and maximum tolerated data delay. Do not treat wall-clock
  polling time as a unique record ID.
- Bound page size and request concurrency. Honor `Retry-After` and rate limits;
  use capped exponential backoff with jitter for retryable transport/server
  failures. Treat authentication/authorization errors as explicit unhealthy
  state, not endless retries.
- Use TLS and an approved credential source. Redact authorization headers,
  query tokens, and response bodies from logs.
- If the API has no stable cursor or replay endpoint, document the resulting
  loss/duplicate window and require an explicit delivery-contract exception.

## Kafka source

- Use a stable consumer group identity for the logical pipeline and source.
  Configure explicit topic and partition ownership; do not run independent
  consumers with conflicting ownership assumptions.
- Disable automatic offset commit. Commit the next offset only after all
  preceding records in that partition have durable ingest-publish
  confirmations. If processing concurrently, maintain a contiguous completed
  offset watermark; never commit across an unfinished gap.
- Use broker-managed group rebalancing, but couple partition revocation to
  drain: stop fetching, finish or abandon in-flight publishes safely, commit
  only contiguous completed offsets, then relinquish ownership.
- Preserve topic, partition, offset, key, and timestamp as source metadata.
  The source identity key used for event deduplication must remain stable
  across rebalances and replay.
- On broker or ingest-lane outage, pause partitions before bounded buffers
  fill. Resume from committed offsets after recovery; expect duplicate
  publication for records fetched but not committed.
- Select `offset_reset` only for partitions without a valid committed offset.
  `earliest` may replay retained history and is not a substitute for recovery
  planning; `latest` may skip history. Require an explicit operator policy for
  a missing/out-of-range committed offset.

## MQTT source

- Use QoS 1 or another explicitly reviewed delivery mode when replayable
  at-least-once delivery is required. QoS 0 can lose events and must not be
  described as at-least-once.
- Use a broker-supported shared subscription for multi-node consumers and
  verify its behavior under disconnect, reconnect, and rebalance. A shared
  subscription distributes messages but does not by itself provide the
  pipeline's durable source cursor or sink exactly-once behavior.
- Use a stable logical client identity where session continuity is required;
  define clean-start/session-expiry settings with the broker operator. Do not
  allow every restart to silently discard a persistent session or create
  unbounded stale sessions.
- Acknowledge a delivery to the broker only after durable ingest publish. If
  the chosen client/broker cannot defer protocol acknowledgement until then,
  document the loss window and do not claim the common guarantee.
- Derive a stable event ID from a producer ID plus producer sequence or another
  immutable application identifier. Topic and payload alone may not uniquely
  distinguish repeated legitimate events.
- Record topic, QoS, packet/session diagnostics, and producer metadata without
  logging sensitive payloads. Treat reconnect storms and expired-session
  re-delivery as observable operational events.

## Shared-file source

- Require shared storage visible to all eligible workers, a deterministic
  listing order, and a distributed claim/lease with owner generation and
  fencing. Node-local file watchers are not safe for shared work assignment.
- Prefer immutable, atomically published files (write temporary name, then
  publish/rename or commit a manifest). Do not read a file while its producer
  is still appending unless a framing and completion protocol exists.
- Identify each record by a stable file identity/version plus record position
  (for example line number only when files are immutable). Persist per-file
  progress only after corresponding records are durably published.
- After a crash, resume from the last durable record progress or replay the
  immutable file using deterministic event IDs. Do not mark a file complete
  on local read completion alone.
- Archive or move a file only after all records are published and completion
  metadata is durable. Use conditional operations or manifests to prevent
  concurrent workers from claiming or archiving the same input.
- Define behavior for partial files, duplicate names, replacement at the same
  path, checksum mismatch, malformed records, permission errors, and storage
  outages. Quarantine is access-controlled and retention-bounded.

## Operational acceptance

Before enabling a source, demonstrate:

- outage and restart replay without cursor/offset gaps;
- duplicate handling after a publish succeeds but source acknowledgement is
  lost;
- drain/rebalance with in-flight records and stale-owner fencing;
- bounded memory/storage use during prolonged backpressure;
- rate-limit, authorization, malformed input, and expired-cursor behavior;
- source-specific lag and health telemetry with alert routing.

Capture source-specific objectives in
[`service-objectives.md`](operations/service-objectives.md) and use the
failure scenarios in [`failure-drills.md`](operations/failure-drills.md).
