# Distributed Failure Drills and Acceptance Criteria

This is a test plan, not an implemented chaos tool or production runbook. Run
drills only in an isolated environment with approved owners, bounded blast
radius, test data, and a verified rollback/stop procedure.
For disaster-recovery restore sequencing, use
[`disaster-recovery-runbook.md`](disaster-recovery-runbook.md); capture each
exercise with [`recovery-drill-record.template.md`](recovery-drill-record.template.md).
For full-node rolling-upgrade rehearsals, use
[`full-node-upgrade-runbook.md`](full-node-upgrade-runbook.md).

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
| Heartbeat jitter / delay | Delay or drop a bounded number of heartbeat requests, then restore delivery | Node suspicion is observable; no overlapping valid owners; lease safety margin and fencing behavior match policy |
| Stale-generation reconnect | Restart a node and replay a heartbeat/ack from its previous generation | Old generation is rejected and cannot renew leases, acknowledge plans, or commit writes |
| Control-plane leader change | Stop the active controller while quorum remains, then restore it | One authoritative plan/epoch history continues; no duplicate assignment or reset epoch |
| Scheduler stale capacity | Freeze or corrupt a non-production worker capacity report | Scheduler marks it stale/ineligible per policy and does not over-assign it |
| Scheduling hot partition | Generate skewed keys with one dominant partition | Hot-partition lag is surfaced; rebalance does not claim extra parallelism or break key order |
| Rebalance drain timeout | Hold an old worker in-flight beyond its bounded drain deadline | Old epoch is fenced before replacement commits; recovery replays from last durable progress |
| Cross-node barrier participant loss | Remove a worker while a declared global barrier is pending | Barrier remains incomplete or follows explicit abort/partial policy; checkpoint does not cross unresolved boundary |
| Coordinator/store outage | Make coordination quorum/store unavailable in an isolated test | No new assignments or unsafe barrier completion; current owners obey lease/fencing safety |
| Network partition and stale writer | Isolate old owner from control plane but retain data-service path, then attempt commit after reassignment | Storage/sink rejects stale epoch; no dual writes; alert and audit evidence are present |

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
- For coordination/scheduling drills, also record heartbeat interval and lease
  policy version, missed-heartbeat sequence, plan generation, old/new
  assignment epochs, placement decision reason, participants, barrier ID, and
  stale-write rejection evidence.

A drill passes only when all safety and service objectives are met and results
are reviewed. Any split-brain write, acknowledged data loss, secret exposure,
unbounded retry, or inability to stop the drill is an immediate failure and
blocks production rollout until remediated and retested.
