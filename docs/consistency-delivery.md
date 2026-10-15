# Network Partitions, Split-Brain Safety, and Delivery Semantics

This document defines design constraints, not implemented guarantees. The
actual behavior depends on the chosen coordination service, message bus, state
store, and sink capabilities.

## Network-partition policy

Prioritize single-writer safety over availability for partition ownership and
state commits:

- Only the side that can prove current control-plane authority and a valid,
  unexpired fencing epoch may change assignment ownership.
- A worker that cannot renew/verify its lease must stop state writes,
  checkpoint advancement, and acknowledgements before the configured safety
  margin expires.
- A control-plane minority must not issue new assignments or advance epochs.
  Existing workers may continue only if their leases and downstream fencing
  remain verifiable under the selected consistency model.
- If the message bus, state store, or checkpoint store cannot validate the
  current epoch, fail closed for the affected partitions.
- Do not infer that a node is dead from one missed heartbeat. Suspect status is
  advisory; lease expiry plus fencing establishes safe reassignment.

The key trade-off is explicit: during a partition, some partitions may pause
until authority is re-established. Do not allow both sides to accept writes in
order to preserve availability.

## Split-brain fencing protocol

1. Store the authoritative assignment epoch in a quorum-backed coordination
   system using compare-and-set.
2. Bind every write-capable assignment token to cluster, source, partition,
   owner node, node generation, and epoch.
3. Have every state/checkpoint commit boundary reject tokens older than its
   highest accepted epoch. Sink writes must use an idempotency key or a sink-side
   fencing mechanism where stale output is harmful.
4. On owner loss, first advance/fence the old epoch, then grant the new owner.
   Never rely on a control API response alone if downstream services do not
   enforce fencing.
5. If the coordination quorum is unavailable, prohibit new assignments and
   preserve the last committed checkpoint. Resume only after quorum recovery
   resolves the authoritative epoch.
6. Audit stale-token rejections and alert on repeated fencing conflicts; do not
   automatically retry a stale writer with the same token.

Wall-clock timestamps may help diagnostics but must not replace monotonic
epochs, compare-and-set, or lease validation. Lease duration and renewal margin
must account for worst-case network delay, scheduler pauses, and clock
uncertainty, then be validated by fault injection.

## Delivery contract

Choose a delivery contract per source-to-sink path and document it in pipeline
configuration before deployment:

| Contract | Required behavior | Consequence |
| --- | --- | --- |
| At-most-once | Acknowledge before processing or do not retry | Events can be lost; use only when loss is acceptable |
| At-least-once | Acknowledge after durable processing/checkpoint | Events can be replayed; state and sinks must be idempotent or deduplicating |
| Effectively-once | At-least-once transport plus stable event identity and durable idempotency/deduplication at every side-effect boundary | Depends on dedupe retention and consistent key semantics |
| Transactional exactly-once | Atomic commit across input position, state, and output | Claim only if the selected systems support and tests prove the complete transaction boundary |

The default planning assumption should be **at-least-once with idempotent
processing**, not exactly-once. Duplicate delivery is expected after ambiguous
timeouts and crashes. Define the event identity, idempotency key, dedupe
retention, ordering scope, and behavior for poison events.

## Checkpoint and replay rules

- A checkpoint records a contiguous committed input position per partition,
  assignment epoch, state version, schema version, and processing version.
- Never checkpoint past an uncommitted gap in an ordered partition.
- Commit state and sink effects before advancing the checkpoint/acknowledging
  input, unless a verified atomic transaction covers them together.
- On retry, use the same stable idempotency key. Treat sink timeout after a
  possible write as an unknown outcome; query/reconcile or retry idempotently.
- Replay only from the last durable committed position. Preserve enough source
  or bus retention to cover the declared recovery window.
- Quarantine poison events with source identity, partition, offset/event ID,
  schema version, and failure class; redact payload secrets and define a
  controlled re-drive procedure.
- A re-drive is a new audited operation. Preserve original identity for
  deduplication while recording the re-drive attempt separately.

## Failure behavior matrix

| Condition | Continue? | Safe action |
| --- | --- | --- |
| Worker loses control-plane connectivity, data path still works | No commits beyond verified lease margin | Pause affected partition; do not acknowledge input |
| Control-plane quorum lost | No membership/ownership mutation | Keep last known assignments only within proven fencing/lease rules; otherwise pause |
| Old and new owner both appear active | Never allow both to commit | Reject stale epoch at state, checkpoint, and sink boundaries |
| State write succeeds, checkpoint write fails | Retry/reconcile | Replay may repeat state operation; require idempotency/transaction |
| Sink write may have succeeded but response is lost | Outcome is unknown | Reconcile or retry with the exact same idempotency key |
| Checkpoint advances but sink effect is not durable | Invalid commit order | Treat as data-loss bug; block release |
| Source retention expires before recovery | Cannot guarantee replay | Escalate as RPO breach; restore from independent archive if available |

## Required partition tests

- Bidirectional network partition between worker and control plane while both
  retain broker access.
- Control-plane quorum loss with delayed/duplicated heartbeats.
- Old worker resumes after a new epoch is issued and attempts state, checkpoint,
  broker, and sink commits.
- Crash at each boundary between input read, state write, sink write,
  checkpoint, and acknowledgement.
- Replay after ambiguous sink timeout with duplicate-event measurement.
- Recovery after source/broker retention is nearly exhausted.

Any concurrent valid writes for one ordered partition, acknowledgement past
durable progress, or silent loss is a blocking correctness failure.
