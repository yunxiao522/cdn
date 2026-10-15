# Performance Benchmarks, Capacity, and Service Objectives

This is a planning template. No benchmark harness, service, or measured results
are included in the repository. Do not treat example formulas or empty targets
as a performance claim.

## Define objectives first

Record measurable service objectives before choosing worker counts. Fill in
targets from workload and business requirements; do not infer hard-real-time
guarantees from low average latency.

| Measure | Target | Measurement window | Notes |
| --- | --- | --- | --- |
| End-to-end event latency (p50/p95/p99/max) | TBD | TBD | Include source, queue, compute, and sink time |
| Sustained throughput (events/s and bytes/s) | TBD | TBD | State event-size distribution |
| Burst throughput and duration | TBD | TBD | Define acceptable queue growth |
| Event loss / duplicate rate | TBD | TBD | State delivery semantics |
| Recovery time after worker restart | TBD | TBD | Include replay and rebalancing |
| Resource ceiling (CPU, memory, disk I/O) | TBD | TBD | Reserve headroom for bursts and recovery |

Report latency percentiles and maxima alongside throughput. Averages alone
hide queueing and tail-latency behavior. Identify cluster size, node hardware
and zones/failure domains, network, OS, runtime, configuration, and source,
broker, state-store, and sink versions.

## Capacity model

For each source and worker group, estimate required processing capacity as:

`required_events_per_second = peak_input_events_per_second × safety_factor`

Estimate retained message storage for a tolerated downstream outage as:

`queue_bytes = peak_bytes_per_second × tolerated_outage_seconds`

Add protocol, serialization, index, replication, cross-zone traffic, and
backup overhead based on measurements. Size each shared service for failure
and recovery headroom, not only steady-state traffic. Worker counts must be
validated experimentally because CPU-bound, I/O-bound, stateful, and
partition-limited workloads scale differently.

Record assumptions per source: average and maximum event size, peak and
sustained arrival rates, partition count, ordering requirement, retention,
downstream outage tolerance, and duplicate/replay behavior.

## Benchmark scenarios

Keep repeatable workload descriptions under `benchmarks/scenarios/`. At minimum
cover:

1. Steady-state throughput at representative event sizes.
2. Burst input and bounded-queue backpressure.
3. Slow or unavailable sink with broker lag growth and recovery.
4. Worker and node loss, partition rebalance, checkpoint restore, and replay.
5. Skewed partition keys and hot-partition behavior.
6. Large payloads and malformed-message quarantine.
7. Broker/state-store failover and cross-failure-domain network latency.
8. Long-running retention, compaction, and storage-pressure behavior.

Run a baseline, change one factor at a time, and preserve machine/configuration
details with every result. Store generated results outside source control unless
they are sanitized, reproducible artifacts explicitly intended for review.

## Acceptance and operational guardrails

- Define pass/fail thresholds for each objective before running a benchmark.
- Measure backpressure, queue depth, processing lag, retries, dropped events,
  replication lag, rebalance time, network/storage use, CPU, memory, and
  p50/p95/p99/max latency.
- Verify that throughput gains do not violate ordering, durability, recovery,
  or duplicate-delivery requirements.
- Repeat after material changes to runtime, schema, source adapter, sink, or
  storage configuration.
- Do not claim hard real-time behavior without worst-case timing analysis and
  verification on the specified OS and hardware.

## Horizontal scaling decision policy

Use separate policies for each worker group (`ingest`, `compute`, and `egress`).
Choose thresholds and windows from load tests; the placeholders below are
required deployment inputs, not default recommendations.
Use the cluster sizing worksheet at
[`../deployments/topologies/production-capacity.template.yaml`](../deployments/topologies/production-capacity.template.yaml)
to capture the reviewed capacity and autoscaling inputs.

| Policy input | Required value |
| --- | --- |
| Minimum and maximum replicas per role | `TBD` |
| Scale-out trigger metric and threshold | `TBD` |
| Trigger duration / evaluation window | `TBD` |
| Scale-in trigger and longer stabilization window | `TBD` |
| Maximum replicas changed per evaluation | `TBD` |
| Cooldown after scaling or rebalance | `TBD` |
| Minimum healthy replicas per failure domain | `TBD` |
| Maximum partitions moved per rebalance | `TBD` |
| Drain and checkpoint deadline | `TBD` |

Decision sequence:

1. Scale out only when sustained lag or saturation breaches its threshold for
   the configured window and there is unused partition parallelism.
2. Check broker, state-store, checkpoint-store, sink, and network health first.
   Do not add workers to mask a shared dependency bottleneck.
3. Add no more than the configured step limit, then wait for assignment,
   restore, and rebalance to finish before another decision.
4. Scale in only after a longer low-load window, while preserving minimum
   replicas per failure domain. Drain, checkpoint, and fence ownership before
   terminating workers.
5. Freeze automated scaling during dependency incidents, control-plane
   uncertainty, active recovery, or repeated rebalance failures; alert an
   operator instead.
6. Roll back or hold capacity if tail latency, errors, duplicate delivery,
   state-store pressure, or rebalance duration worsens after a change.

Never scale workers beyond the number of independently processable partitions
expecting linear throughput. Define metric freshness, missing-metric behavior,
cooldown, and a manual override before enabling an autoscaler.
