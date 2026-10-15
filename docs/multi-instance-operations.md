# Multi-Instance Throughput and Availability Guide

This guide turns the example topology into a reviewable multi-instance
operating model. The example counts are not a production recommendation,
availability guarantee, or latency/throughput claim. No runtime or autoscaler
is implemented in this repository. Select targets from workload measurements
and record them in
[`../deployments/topologies/production-capacity.template.yaml`](../deployments/topologies/production-capacity.template.yaml).
For a comparison with the centralized cluster mode and migration path, see
[`architecture-modes.md`](architecture-modes.md).

## Replica count is not capacity

For each worker role, capacity is limited by the minimum of:

- independently processable source/message partitions;
- per-partition throughput before latency or resource limits are breached;
- worker CPU, memory, network, and storage capacity;
- the sustainable capacity of shared state, checkpoint, broker, source, and
  sink dependencies.

Adding replicas beyond useful partition parallelism does not increase
throughput. It may increase coordination, idle resource use, and rebalance
work. A hot partition remains a bottleneck unless the business ordering
contract permits changing its key or splitting its work.

Use measurements to estimate a steady-state role capacity:

`role_capacity ≈ active_partitions × measured_safe_rate_per_partition`

Then test node/zone loss and recovery load separately. Normal traffic capacity
must leave room for failover, catch-up replay, and rolling maintenance. Do not
derive a replica count from CPU averages or event counts alone; include event
size distribution, state cost, sink latency, burst duration, and tail-latency
objectives.

## Placement and independent failure

- Place control-plane quorum members across independent failure domains
  according to the selected coordination system's quorum rules. A replica
  count alone does not provide quorum availability if all members share one
  failure domain.
- Spread ingest, compute, and egress replicas so a single node/domain loss
  does not remove every healthy member of a required role.
- Treat the broker, coordination, state, checkpoint, identity, and telemetry
  services as separately sized and placed dependencies. Application replicas
  cannot compensate for a saturated or unavailable shared dependency.
- Define anti-affinity, topology spread, disruption budgets, resource
  requests/limits, and graceful termination in the selected deployment
  platform. Ensure these settings leave enough capacity to drain and reschedule
  workers during a failure.
- Keep stateless connector/worker replicas independently replaceable. Store
  authoritative offsets, ownership, state, and checkpoints only in their
  approved shared durable services.

## Parallelism and memory bounds

For each worker role, record the maximum active partitions per replica, maximum
in-flight records, maximum batch size/bytes, and memory budget. Validate the
combined upper bound, including serialization copies, client buffers,
checkpoint/state cache, retry buffers, and runtime overhead, against the
container/process limit. A nominal record count does not bound memory when
event size is unbounded.

Set explicit maximum event size, queue retention, in-flight count, batch bytes,
and retry age. Enforce payload limits at ingestion and define an explicit
quarantine/rejection path. Apply backpressure before local memory or temporary
storage is exhausted. If an upstream source cannot pause, calculate a bounded
durable spool size and expiry behavior from the measured peak byte rate and
maximum dependency outage.

Use separate worker pools and resource budgets for ingest, compute, and egress
when one role's backlog must not starve another. Stateful processing must
include restore time and state-store throughput in partition placement and
rebalance decisions.

## Scaling controls

Configure scale-out and scale-in independently for ingest, compute, and egress.
Use sustained, sufficiently fresh signals (source lag/age, consumer lag,
delivery lag, saturation, and dependency health), not a single CPU or queue
sample. All thresholds, windows, replica bounds, step sizes, and cooldowns
remain deployment-specific inputs.

Before scale-out:

1. Confirm the relevant source/message partitions can be assigned to additional
   workers.
2. Check broker, state, checkpoint, sink, network, and control-plane health.
   Do not scale out to mask a dependency outage.
3. Verify the new capacity will not exceed downstream quotas or shared-store
   limits.
4. Add a bounded number of replicas, then wait for assignment, state restore,
   and rebalance completion before making another decision.

Before scale-in:

1. Confirm load is below a reviewed threshold for the longer stabilization
   window and no incident/recovery freeze is active.
2. Drain selected owners, stop new intake, finish or safely replay in-flight
   work, and commit only contiguous durable progress.
3. Fence the old assignment epoch before transferring ownership.
4. Remove capacity only after replacement owners are ready and service
   objectives remain healthy.

Missing or stale metrics, control-plane quorum loss, lease uncertainty,
dependency degradation, repeated rebalance, or active disaster recovery must
freeze automatic scaling and raise an actionable alert. Manual override and
resume criteria must be documented and audited.

## High-throughput acceptance profile

Before describing a deployment as high-throughput or low-latency, measure and
approve:

| Test dimension | Required evidence |
| --- | --- |
| Steady load | Sustained events/s and bytes/s with p50/p95/p99/max end-to-end and stage latency |
| Burst and skew | Burst duration, maximum queue growth, hot-partition rate, and time to drain |
| Replica scaling | Throughput/latency per replica and the point where partition or dependency limits dominate |
| Failure headroom | Results with one worker/node/failure domain unavailable, including recovery replay |
| Backpressure | Memory/spool upper bound and proof of no silent loss when the sink or broker slows |
| Rebalance | Duration, moved partitions, state restore time, duplicate/replay behavior, and objective impact |
| Long run | Resource growth, connection counts, checkpoint/retention behavior, and telemetry cardinality |

Repeat tests after changes to event schema/size, partition count/key, runtime,
worker limits, broker/state/sink versions, or retry/batch policy. A deployment
passes only if the declared throughput, tail-latency, delivery, and recovery
objectives all hold together.

## Operational guardrails

- Use readiness to gate traffic/ownership until dependencies and assignment
  state are usable; liveness must not trigger restart loops for a recoverable
  shared dependency outage.
- Bound restart rates, retry budgets, rebalance batches, and drain deadlines.
  Escalate when a limit is reached rather than looping indefinitely.
- Alert on oldest-event age and lag growth, not only process health.
- Do not claim hard-real-time behavior from replica counts, autoscaling, or
  successful average-latency tests. This design targets low-latency processing
  on general-purpose infrastructure; deadlines and worst-case latency require
  a separately specified and verified system.

See [`performance-capacity.md`](performance-capacity.md) for workload models
and scaling policy inputs,
[`runtime-architecture.md`](runtime-architecture.md) for assignment/drain
flows, and [`observability.md`](observability.md) for required signals.
For shared-cluster tenant quotas and fairness, see
[`tenant-isolation-and-admission.md`](tenant-isolation-and-admission.md).
Node heartbeat/lease safety is specified in
[`node-communication.md`](node-communication.md); placement and load balancing
are specified in
[`task-scheduling-and-load-balancing.md`](task-scheduling-and-load-balancing.md).
For co-locating control-plane participation and data-plane capability on every
node, see [`full-node-cluster.md`](full-node-cluster.md).
