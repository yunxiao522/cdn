# Multi-Node Cluster Bootstrap and Shutdown Runbook

This is a provider-neutral operational sequence, not an executable procedure.
The repository has no cluster launcher, node agent, connector, or deployment
manifest. Replace every `TBD`, map the steps to approved platform tooling, and
test in an isolated environment before production use. Do not infer production
readiness from the example topology.
For the full-node variant, also preserve control quorum and resource isolation
while starting or draining data-plane work; see
[`../full-node-cluster.md`](../full-node-cluster.md).

## Bootstrap record

- Cluster/environment and change ID: `TBD`
- Operator / approver / incident contact: `TBD`
- Target control-plane and worker generations: `TBD`
- Topology/config/schema/plan versions: `TBD`
- Change window and rollback deadline: `TBD`
- Approved source/sink canary scope: `TBD`
- SLO, RPO/RTO, lag and observation-window thresholds: `TBD`
- Stop/abort authority and out-of-band channel: `TBD`

Use the release record at
[`release-readiness.template.md`](release-readiness.template.md) for a planned
rollout. For disaster recovery, follow
[`../operations/disaster-recovery-runbook.md`](../operations/disaster-recovery-runbook.md)
instead of treating this bootstrap sequence as a restore procedure.

## Preconditions

- [ ] A selected orchestrator, node inventory, failure-domain placement, and
  capacity plan are approved.
- [ ] Control-plane quorum, message bus, coordination store, state store,
  checkpoint store, identity/secrets, and telemetry dependencies are provisioned
  and have provider-specific readiness checks.
- [ ] Network paths, DNS, time synchronization, TLS, workload identities,
  least-privilege policies, and secret rotation have been verified.
- [ ] Topic/stream lanes, schema versions, retention, partition counts,
  checkpoints, backup manifests, and output idempotency behavior are reviewed.
- [ ] Source cursor/offset policies and sink limits are set. All retry,
  in-flight, event-size, queue, and batch bounds have approved values.
- [ ] No unexpected old cluster owner is active. Current membership, plan
  generation, assignment epochs, and source progress have been recorded.
- [ ] Dashboards, alerts, runbooks, operator access, and telemetry redaction
  have been tested.
- [ ] A verified rollback/abort plan preserves durable offsets, data, and
  monotonic fencing epochs.

If any safety-critical value remains `TBD`, do not enable production traffic.

## Start sequence

### 1. Validate external dependencies

1. Check each dependency using its authoritative health/consistency mechanism,
   not only DNS or an open TCP connection.
2. Confirm broker retention and partition availability, coordination-store
   quorum, state/checkpoint read/write and fencing behavior, and sink
   idempotency/reconciliation.
3. Verify backup manifests and restore points are readable. Do not restore
   over live state during ordinary startup.
4. Confirm telemetry export and alert delivery; telemetry failure must be
   visible and must not be interpreted as healthy.
5. Abort if a required dependency is degraded or cannot enforce the required
   durability/fencing contract.

### 2. Establish the control authority

1. Start the approved control-plane replica set using the selected platform's
   audited procedure.
2. Wait for quorum and confirm there is one authoritative cluster identity and
   monotonic epoch history.
3. Verify the control API's liveness/readiness, identity validation,
   authorization, audit logging, and ability to read/write coordination state.
4. Confirm no assignment or plan generation is advanced by a minority or
   conflicting controller.
5. Keep source intake and worker assignments disabled during this phase.

### 3. Register data-plane nodes

1. Start node agents/workers with authenticated cluster, node, generation, and
   role identities.
2. Register each node and verify the control plane accepted its exact
   generation, capabilities, failure domain, resource limits, and protocol
   version.
3. Confirm heartbeats and dependency readiness. A live process without usable
   dependencies must not become eligible for ownership.
4. Confirm only one active generation is associated with each workload slot;
   fence stale generations before proceeding.
5. Compare observed membership and placement with the approved inventory.
   Unrecognized nodes or role drift block activation.

### 4. Prepare and assign work

1. Validate the versioned pipeline DAG, schemas, source/sink bindings, tenant
   policy, partition mapping, and resource budgets.
2. Create a plan generation and bounded assignment set. Keep intake paused or
   point sources to a non-production/test lane.
3. Assign a small canary set of partitions with new monotonically increasing
   assignment epochs.
4. Require each new owner to restore a compatible checkpoint, validate state
   and source position, and acknowledge the exact plan generation and epoch.
5. Confirm old owners are drained/fenced and downstream state/checkpoint/sink
   boundaries reject stale epochs before any ownership overlap can write.
6. Activate only the canary partitions. Use larger batches only after the
   canary passes the observation window.

### 5. Enable traffic progressively

1. Enable a bounded source set or rate-limited traffic slice.
2. Verify event schema validation, deterministic IDs/partition keys, durable
   ingest publication, and source acknowledgement ordering.
3. Verify compute checkpoints do not cross gaps or unresolved barriers.
4. Verify sink outcomes, idempotency, output completion records, and retry
   behavior.
5. Compare measured end-to-end latency percentiles, oldest-event age,
   source/consumer/output lag, error/retry/dead-letter rates, resource headroom,
   and assignment churn to approved limits.
6. Expand in bounded batches, waiting for stable health after each batch.
   Record evidence and approval for each expansion.

## Abort and safe stop

Abort rollout and stop adding traffic if quorum or lease safety is uncertain,
stale-owner writes are accepted, checkpoints advance past durable output,
source progress has unexplained gaps, sink outcomes cannot be reconciled, a
declared loss/duplicate budget is breached, or telemetry needed to assess
safety is unavailable.

For an ordinary planned shutdown:

1. Announce the change and stop new source intake using source-native pause or
   an approved admission gate.
2. Keep egress active while already accepted work drains, unless a sink
   incident requires it to pause.
3. Stop compute from accepting new work; finish in-flight work or abandon it
   for replay from the last durable checkpoint. Do not commit across gaps.
4. Persist/verify checkpoints, output intents, completion records, and
   source positions. Resolve ambiguous sink outcomes or leave them explicitly
   unknown for recovery.
5. Drain owners, advance/fence assignments, and confirm stale writes are
   rejected. Release ownership only after the new owner or paused state is
   authoritative.
6. Stop egress after durable output lanes are drained or intentionally left
   retained. Stop ingest workers after source cursors are safely committed.
7. Stop data-plane nodes, then control-plane replicas only when no active
   assignments remain or the approved recovery protocol governs restart.
8. Preserve logs, metrics, traces, offsets, plan version, epochs, and operator
   actions. Verify retention/backup jobs remain safe.

Do not terminate nodes by deleting local PID/lock files or resetting
assignment epochs. If a drain deadline expires, fence the old owner and replay
from the last durable checkpoint; tolerate duplicates under the documented
idempotency contract.

## Go/no-go checks

Proceed to the next rollout stage only when all are true:

- [ ] Control-plane quorum and authoritative plan/epoch state are healthy.
- [ ] Exactly one valid owner exists for each active partition.
- [ ] All active owners have acknowledged their current plan generation and
  assignment epochs.
- [ ] Input/output positions and checkpoints are consistent; no unexplained
  gaps exist.
- [ ] Stale owners are rejected at every write/commit boundary.
- [ ] SLO, lag, error, duplicate, retry, and resource thresholds meet the
  approved canary criteria.
- [ ] Sink effects are confirmed or ambiguous outcomes are explicitly tracked.
- [ ] Alerts and telemetry are current, redacted, and actionable.
- [ ] Operator, approver, timestamp, evidence, and next decision are recorded.

## Completion and restart

At completion, record actual topology, node generations, software/config/schema
versions, plan generation, assignment epochs, source positions, checkpoint
references, canary measurements, deviations, and approvals. A restart must
re-read authoritative membership, plan, and checkpoint state; it must not
assume that the prior process's memory or local files establish ownership.

Run required failure/recovery drills in
[`../operations/failure-drills.md`](../operations/failure-drills.md) before
claiming operational readiness. This low-latency design does not promise
hard-real-time deadlines.
