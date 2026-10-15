# Service Objectives, RPO/RTO, and Alert Policy

This is a fill-in template, not an availability commitment. Set targets from
business requirements and validate them using the benchmark and failure-drill
plans before production. Do not treat `TBD` as a passing threshold.

## Service objectives

| Service indicator | Objective | Window / scope | Owner |
| --- | --- | --- | --- |
| End-to-end event latency p99 | `TBD` | Per pipeline; include source-to-sink | `TBD` |
| Sustained accepted throughput | `TBD` | Per source and cluster | `TBD` |
| Successful delivery ratio | `TBD` | Per sink and rolling window | `TBD` |
| Event loss / duplicate budget | `TBD` | Per delivery contract | `TBD` |
| Consumer lag recovery time | `TBD` | After defined burst/outage | `TBD` |
| Control-plane availability | `TBD` | Monthly; define quorum behavior | `TBD` |
| Worker-group availability | `TBD` | Per role and failure domain | `TBD` |

Document the eligible event population, exclusions, measurement source, and
error-budget policy for every objective. Report tail latency and maximums, not
only averages.

## Recovery objectives

| Failure scope | RPO | RTO | Required recovery path |
| --- | --- | --- | --- |
| Single worker process | `TBD` | `TBD` | Fence, reassign, restore checkpoint, replay |
| Single node | `TBD` | `TBD` | Replace node and rebalance partitions |
| Availability-zone/failure-domain loss | `TBD` | `TBD` | Survive or fail over using replicated services |
| Message-bus partition loss | `TBD` | `TBD` | Broker recovery and replay |
| Shared state/checkpoint outage | `TBD` | `TBD` | Restore durable state and reconcile positions |
| Control-plane quorum loss | `TBD` | `TBD` | Restore quorum; prohibit unsafe reassignment meanwhile |
| Regional/site loss, if applicable | `TBD` | `TBD` | Restore from separately managed backup |

RPO is the maximum acceptable loss of acknowledged progress; RTO is the maximum
time to restore the defined service level. State whether each target is
measured per partition, worker group, or whole cluster.

## Alert policy template

| Signal | Trigger | Severity | Initial response |
| --- | --- | --- | --- |
| End-to-end p99 latency | Above objective for `TBD` consecutive windows | `TBD` | Check source lag, queue, compute saturation, and sink latency |
| Consumer/source lag | Above threshold or growing for `TBD` | `TBD` | Check partition skew, worker capacity, and dependency health |
| Oldest unprocessed event age | Above objective for `TBD` | `TBD` | Identify stalled partitions and pause unsafe autoscaling |
| Assignment/rebalance duration | Above `TBD` or repeated epochs | `TBD` | Check fencing, checkpoint restore, and control-plane health |
| Control-plane quorum/lease health | Quorum lost or lease safety margin breached | `TBD` | Freeze reassignment; fail closed |
| State/checkpoint commit errors | Non-zero sustained errors for `TBD` | `TBD` | Stop progress acknowledgements and protect replayability |
| Sink failure/ambiguous timeout | Rate above `TBD` | `TBD` | Check idempotency, destination state, and retry budget |
| Replication/backup health | Lag or verification failure above `TBD` | `TBD` | Protect data; suspend destructive retention if needed |
| Capacity headroom | Resource or storage forecast below `TBD` | `TBD` | Scale within safe bounds or reduce/shape intake |

Use separate warning and paging thresholds, an explicit evaluation window,
deduplication, routing owner, runbook, and maintenance suppression policy.
Avoid alerts based on a single transient sample. Missing or stale telemetry must
be treated as unknown/degraded, not healthy.

## Review and validation

- Approve objectives with service owners and consumers before launch.
- Validate latency/throughput objectives in
  [`performance-capacity.md`](../performance-capacity.md).
- Validate recovery objectives in
  [`failure-drills.md`](failure-drills.md) and
  [`backup-recovery.md`](backup-recovery.md).
- Review targets after traffic, topology, broker, state-store, or retention
  changes and after every failed recovery exercise.
