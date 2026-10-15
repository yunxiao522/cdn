# Distributed Failure Drills and Acceptance Criteria

This is a test plan, not an implemented chaos tool or production runbook. Run
drills only in an isolated environment with approved owners, bounded blast
radius, test data, and a verified rollback/stop procedure.
For disaster-recovery restore sequencing, use
[`disaster-recovery-runbook.md`](disaster-recovery-runbook.md); capture each
exercise with [`recovery-drill-record.template.md`](recovery-drill-record.template.md).

## Preconditions

- Define the tested cluster version, topology, partitions, workload, and
  expected delivery semantics.
- Set measurable RPO/RTO and latency/lag objectives before each drill.
- Confirm backups and recovery procedures; use non-production credentials.
- Ensure operators can stop fault injection immediately and restore service.
- Capture control-plane events, assignments/epochs, broker offsets, checkpoints,
  sink idempotency keys, logs, metrics, and traces with synchronized time.

## Drill matrix

| Drill | Injected failure | Required pass criteria |
| --- | --- | --- |
| Worker process loss | Terminate one assigned worker during traffic | Old epoch is fenced; partitions are reassigned once; recovery meets RTO; no acknowledged input is lost; duplicates remain within the documented delivery contract |
| Node/network isolation | Block worker-to-control-plane traffic while retaining data-plane access, then reverse | Isolated node stops lease-dependent commits before lease safety margin; no two valid writers; recovery resumes from committed checkpoints |
| Control-plane member loss | Stop one member, then restore it | Quorum behavior matches topology; no conflicting assignments; membership and epoch state converge after recovery |
| Control-plane quorum loss | Isolate enough members to remove quorum in a test cluster | No new ownership changes are committed; existing-owner behavior matches lease policy; service fails closed where authority is uncertain |
| Broker partition outage | Make one message partition unavailable | Affected work pauses and lag/health alerts fire; unaffected partitions continue if designed to; no offset skips |
| State/checkpoint outage | Deny state writes or checkpoint commits | Input is not acknowledged beyond durable progress; retries/replay recover without state divergence |
| Sink ambiguous timeout | Allow sink write but drop its response | Retry uses the same idempotency key or reconciles outcome; no silent data loss and duplicates match documented semantics |
| Zone loss | Remove one failure domain | Required control quorum and service replicas survive; partition ownership is fenced/reassigned; recovery objectives are met |
| Rolling schema change | Run mixed versions against compatible and incompatible events | Compatible versions interoperate; incompatible data is held/quarantined explicitly; rollback or migration path works |
| Capacity/rebalance stress | Increase load or worker count across rebalance | Lag, tail latency, and rebalance duration stay within declared objectives; scaling stops safely when limits are reached |

## Acceptance record

For each run, record:

- Drill ID, date, operator, approval, environment, software/configuration
  versions, and injected fault duration/scope.
- Baseline and observed p50/p95/p99/max latency, input/output lag, duplicate
  and loss counts, recovery duration, and affected partitions.
- Whether RPO/RTO, fencing, authorization, alerting, and rollback criteria
  passed; link relevant sanitized telemetry.
- Follow-up actions with an owner and due date for every failure or unexplained
  result.

A drill passes only when all safety and service objectives are met and results
are reviewed. Any split-brain write, acknowledged data loss, secret exposure,
unbounded retry, or inability to stop the drill is an immediate failure and
blocks production rollout until remediated and retested.
