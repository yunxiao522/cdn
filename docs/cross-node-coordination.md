# Cross-Node Coordination and Multi-Worker Execution

This is a technology-neutral design contract, not an implementation. The
target topology is one logically centralized, quorum-backed control authority
managing multiple data-plane nodes in one cluster. This document separates
cluster coordination (who may run work) from dataflow coordination (how
distributed work progresses).

## Coordination layers

| Layer | Authority / location | Responsibilities | Must not do |
| --- | --- | --- | --- |
| Cluster control plane | Replicated control-plane services backed by a quorum store | Authenticate/register nodes, track generations/leases, assign work, issue epochs, plan drains and rebalance | Proxy event payloads or make per-event decisions on the hot path |
| Node agent / worker runtime | Each data-plane process/node | Renew membership, receive assignments, validate epoch, report health, drain owned work | Invent ownership or continue commits after lease/fencing loss |
| Dataflow coordination | Compute workers plus durable messaging/state/checkpoint services | Coordinate partition-local progress, windows, joins, barriers, and checkpoints required by a pipeline | Assume local memory is shared or require a global round trip for every event |
| Durable data services | Shared broker, state, checkpoint, and output services | Persist events, keyed state, progress, and output intents across node loss | Accept stale-owner writes or claim atomicity they do not provide |

The control plane may have several service replicas for availability, but there
is one authoritative membership and assignment history. A minority partition
must not create a competing authority. A data-plane node may host multiple
roles only when resource isolation, identity, scheduling, and failure impact
are explicitly accepted.

The placeholder directory split is:

```text
src/
├── control-plane/
│   ├── api/                    # Planned external control API
│   ├── cluster-coordination/   # Membership, leases, epochs, cluster events
│   ├── membership/             # Node identity and lifecycle
│   ├── partition-assignment/   # Assignment plans and acknowledgements
│   └── scheduling/             # Placement and bounded rebalance decisions
├── data-plane/
│   ├── coordination/           # Pipeline barriers, windows, joins, progress
│   ├── compute/
│   ├── egress/
│   └── ingest/
└── platform/
    ├── coordination/           # Provider-facing quorum/lease primitives
    ├── checkpointing/
    ├── messaging/
    └── state/
```

These are responsibility boundaries, not package/API commitments. The
platform coordination adapter supplies primitives; the control plane applies
them to cluster membership and assignments; pipeline workers use the dataflow
layer for job-level synchronization.

## Multi-node job execution

Represent a pipeline as a versioned directed acyclic graph (DAG) of stages.
Each edge declares its durable handoff, schema version, partition key, ordering
scope, and delivery contract. Each stateful stage declares keyed state,
parallelism/partition compatibility, checkpoint inputs, and recovery behavior.
For every stage, record eligible node roles, resource bounds, and dependency
health requirements before scheduling.

Execution sequence:

1. Validate the pipeline graph and all source/sink contracts; reject cycles
   unless the runtime explicitly models a bounded feedback loop.
2. Resolve each stage to a worker group and eligible nodes based on capability,
   resource policy, tenant quota, and failure-domain constraints.
3. Create a versioned execution plan with pipeline ID, plan generation, stage
   IDs, partition mapping, dependencies, and resource bounds.
4. Allocate partition ownership under monotonically increasing assignment
   epochs. The coordinator stores the authoritative plan; each worker receives
   only its authorized assignments.
5. Workers prepare dependencies and restore compatible state before
   acknowledging readiness for the exact plan generation and epoch.
6. Activate the plan only after required stages are ready. Route data through
   durable stage boundaries; workers commit progress according to the
   per-partition delivery/checkpoint contract.
7. On plan change, stop new work for affected partitions, drain or fence old
   owners, prepare replacements, then switch routing at a verified checkpoint.
8. Persist plan version, stage version, schema version, source position, and
   assignment epoch with recovery metadata so replay is deterministic or
   explicitly version-migrated.

Workers must reject stale plan generations as well as stale assignment epochs.
Generation and epoch have distinct purposes: plan generation identifies the
pipeline graph/configuration; assignment epoch fences ownership of a
partition. A new plan must not reuse or roll back assignment epochs.

## Cross-partition synchronization

Prefer partition-local progress and durable asynchronous handoffs. Introduce
cross-partition barriers only where semantics require them, such as aligned
window completion, consistent global snapshots, or a multi-partition cutover.

- Give each barrier a unique pipeline/stage/plan/window-or-checkpoint ID and
  record the exact participant partition set.
- Record arrival against each partition's last durable sequence/offset and
  current assignment epoch. Reject stale or duplicate arrivals idempotently.
- Complete only when the declared participant set has arrived or an explicit
  failure/timeout policy resolves missing participants. Never silently shrink
  the set to make a barrier appear successful.
- Persist barrier state durably if it must survive coordinator/node restart.
  A barrier held only in process memory cannot be used as a recovery boundary.
- Bound barrier wait time and pending-barrier count; expose oldest wait and
  blocked partitions. On timeout, pause/fail the affected stage or use a
  documented partial-window policy; do not advance checkpoints across an
  unresolved consistency boundary.
- Avoid a cluster-wide lock or synchronous all-node acknowledgement per event.
  Batch/control-plane coordination and data-plane event delivery have separate
  paths and budgets.
- If partitions are added/removed during a barrier, finish or abort the old
  participant set explicitly and create a new barrier under the new plan
  generation.

Global barriers reduce availability to the slowest participant and may create
head-of-line blocking. Benchmark their frequency, participant count, wait
distribution, and recovery behavior. Do not describe barrier-based processing
as low-latency until measured end to end.

## Scheduling and placement

The centralized scheduler chooses placement but should not micromanage each
record. Scheduling decisions should use:

- node role/capability, free CPU/memory/network budget, and health;
- partition locality, state restore cost, and current ownership;
- failure-domain spread and minimum surviving capacity;
- tenant quota/fairness and stage priority;
- broker/state/sink capacity and maximum rebalance work.

Use bounded plan sizes and move only a controlled set of partitions per
rebalance. Preserve key ordering and state locality where required. If no safe
eligible owner exists, leave the partition unassigned/paused and raise an
actionable alert rather than issuing a conflicting assignment.

## Failure behavior

| Condition | Required response |
| --- | --- |
| Control-plane quorum unavailable | Do not commit new cluster membership/assignment decisions; existing workers continue only while their current leases and downstream fencing remain provable |
| Worker isolated from control plane | Stop lease-dependent writes before the safety margin expires; do not self-elect or assume ownership |
| Worker crashes during a barrier | Keep the barrier incomplete; fence/reassign the partition and restore its durable arrival/progress state, or abort the barrier under its explicit policy |
| Plan update races with rebalance | Reject stale plan-generation acknowledgements; finish or abort the old plan transition before activating the new one |
| Shared coordination store unavailable | Fail closed for ownership mutations and barriers that require durable state; do not substitute local locks |
| One partition is slow or hot | Isolate/report the lag; do not skip it in a global barrier unless partial progress is explicitly allowed |
| Data service rejects stale epoch | Stop the stale worker, audit the rejection, and recover from the latest durable checkpoint |

Cluster membership and assignment semantics are specified in
[`cluster-protocol.md`](cluster-protocol.md); event commit boundaries are in
[`processing-lifecycle.md`](processing-lifecycle.md); split-brain policy is in
[`consistency-delivery.md`](consistency-delivery.md).

## Required tests and operational signals

Test quorum loss, delayed/reordered heartbeats, stale plan and assignment
messages, coordinator restart during a barrier, worker loss at each barrier
boundary, partition add/remove during plan rollout, and state restore under a
new owner. Verify one valid writer per partition and no checkpoint advancement
past an unresolved barrier.

Measure plan creation/activation time, assignment churn, barrier duration and
timeouts, participant count, blocked partitions, stale-generation/epoch
rejections, restore duration, and per-stage queue age. Alert on control-plane
authority loss, conflicting owners, long-running barriers, and repeated plan
activation failures. Keep metric labels bounded as defined in
[`observability.md`](observability.md).

## Performance boundary

The architecture targets measurable low-latency distributed processing on
general-purpose infrastructure. It does not promise hard-real-time deadlines
or “ultra-high” throughput. Set latency/throughput objectives and verify
percentiles, maximums, overload, skew, failure, and recovery using
[`performance-capacity.md`](performance-capacity.md) and
[`multi-instance-operations.md`](multi-instance-operations.md).
