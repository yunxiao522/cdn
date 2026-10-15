# Event Contract, Topics, and Partitioning

This is a proposed cross-service contract; no running producer, broker, schema
registry, or validator enforces it yet. The example envelope schema is
[`src/contracts/schemas/v1/event.schema.json`](../src/contracts/schemas/v1/event.schema.json),
and logical topic lanes are illustrated in
[`config/examples/topics.example.yaml`](../config/examples/topics.example.yaml).

## Event envelope

Every event crossing a service boundary should include:

| Field | Rule |
| --- | --- |
| `event_id` | Stable across retry/replay; used for deduplication and audit |
| `event_type` | Lowercase logical event name, independent of topic name |
| `schema_version` | Explicit major contract version |
| `occurred_at` | Producer event time in RFC 3339; ingestion time is separate transport metadata |
| `producer.id` / `producer.version` | Identifies producer software without embedding credentials |
| `partition_key` | Stable key for events that require per-entity ordering |
| `payload` | Type-specific data validated against its versioned contract |
| `trace` | Optional distributed trace context; never place secrets here |

Broker offset, ingestion timestamp, assignment epoch, retry count, and delivery
attempt belong in transport/processing metadata, not the immutable event
payload. Preserve the original `event_id` and payload on retry; increment
attempt metadata separately. Do not use timestamps or random partitioning as a
substitute for a stable idempotency key.

## Logical topic lanes

Use environment-, domain-, pipeline-, lane-, and major-version-qualified names:

`<environment>.<domain>.<pipeline>.<lane>.v<major>`

Recommended logical lanes are:

1. `ingest.raw`: validated source envelope before transformations.
2. `compute.normalized`: canonical normalized event.
3. `egress.results`: durable output intents/results for destinations.
4. `compute.retry`: retryable failures, with bounded attempts and delayed
   re-delivery policy.
5. `compute.dead-letter`: malformed, poison, or exhausted events for audited
   inspection and controlled re-drive.

Map logical lanes to the selected broker's native topics, streams, or queues.
Apply least-privilege ACLs by producer/consumer role. Dead-letter payloads may
contain sensitive data; redact or encrypt them, set explicit short retention,
and limit operator access.

## Partition and ordering policy

- Choose `partition_key` from the business entity whose events must be ordered.
- Hash/route the same key consistently at every lane where per-key order is
  required. Preserve the key during normalization and retries.
- Ordering is only promised within one partition/key; do not imply global order.
- Partition count is a capacity and compatibility decision. Increasing it can
  remap keys and affect ordering, state locality, and consumer assignments;
  plan a versioned migration or an explicit broker-supported strategy.
- Avoid high-cardinality-per-tenant hot keys and null/empty keys. Define a
  deterministic fallback or reject/quarantine before publishing.
- For Kafka, ensure consumer parallelism does not exceed useful partition
  parallelism. Other brokers must provide equivalent documented ownership
  behavior before claiming ordered processing.
- Retry/dead-letter re-drive can interleave with newer events. If strict
  per-key order matters, pause that key/partition or use a sequenced re-drive
  process.

## Schema evolution

- Add optional fields compatibly; do not silently change field meaning or type.
- Version breaking changes with a new major schema/topic lane and use an
  explicit migration or dual-read/dual-write window.
- Retain schema and processing versions in checkpoints and audit metadata so
  replay uses compatible logic.
- Validate at ingress and again at trust boundaries. Invalid events must not be
  silently dropped; quarantine with a bounded, redacted diagnostic.
- Test old/new producer and consumer combinations before rollout.

## Delivery and replay metadata

The default delivery assumption is at-least-once. Consumers must be idempotent
on `event_id` or a domain-specific idempotency key. Track source position,
partition, assignment epoch, attempt, schema version, and failure class as
separate metadata. A re-drive preserves event identity, records the operator
and reason, and is measured as replay rather than new source traffic.
