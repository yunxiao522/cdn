# Full-Node Rolling Upgrade and Protocol Migration Runbook

This is a provider-neutral change procedure, not an executable deployment
script. It applies to the full-node profile where each node can host control
participation and data-plane workers. Use it only after adapting it to the
selected consensus provider, orchestrator, data services, and approved release
process.

## Upgrade invariants

- Preserve quorum and one authoritative control history for the entire change.
- Never allow a node with an older term, plan generation, or assignment epoch
  to commit after a newer owner is active.
- Maintain at most one active writer per ordered partition.
- Do not roll back monotonic terms, fencing epochs, source offsets, or committed
  checkpoints.
- Do not migrate stored state or event meaning irreversibly until the
  forward-compatibility and recovery paths are proven.
- Keep sufficient data-plane capacity to meet the approved SLO and to recover
  partitions while nodes are drained.
- If version compatibility or fencing cannot be proven, pause the affected
  work and stop the rollout rather than running an unsafe mixed version.

## Upgrade record

- Change/release ID and source revision: `TBD`
- Old and target artifact digests: `TBD`
- Old and target control protocol versions: `TBD`
- Old and target plan/config/schema/state versions: `TBD`
- Consensus provider and quorum/failure tolerance: `TBD`
- Nodes and failure domains in scope: `TBD`
- Maximum concurrent unavailable control participants: `TBD`
- Data-plane capacity and rebalance budget during rollout: `TBD`
- Canary partitions, bake window, and SLO guardrails: `TBD`
- Operator, approver, stop authority, and rollback/forward-fix owner: `TBD`

Do not proceed while a safety-critical value is `TBD`.

## Compatibility classification

Before building a sequence, classify each change:

| Change | Typical safe approach | Stop condition |
| --- | --- | --- |
| Additive control API field | New readers accept old messages; old readers ignore the optional field only if protocol explicitly permits it | Field changes authorization, ownership, or fencing semantics |
| Control protocol version | Expand support first, then roll participants, then contract/remove old support in a later release | No supported mixed-version window or old participant can mutate incompatible state |
| Event schema additive field | Deploy tolerant readers before producers emit it | Required field or semantic change breaks retained data/replay |
| State/checkpoint format | Versioned dual-read or staged migration with verified backup/restore | Migration cannot be reversed or old binary cannot safely read committed state |
| Partition key/count/routing | Explicit new plan generation and migration; preserve single-writer ownership per partition | Key remap or count change would silently reorder or lose state |
| Assignment epoch/fencing | Coordinated protocol rollout with downstream enforcement already deployed | Any path can accept stale owner writes |
| Secret/identity rotation | Bounded overlap, verify new identity, then revoke old identity | Workers fall back to static/unauthenticated credentials |

“Backward compatible” must be demonstrated for both live traffic and retained
messages/checkpoints used during recovery.

## Preflight gates

- [ ] Release readiness record is complete and approved.
- [ ] Artifact signatures/digests, provenance, configuration, schemas, and
  dependency versions are verified.
- [ ] Mixed-version matrix covers control participants, node agents, workers,
  APIs, schemas, checkpoints, brokers, state stores, and sinks.
- [ ] Restore test proves the candidate can read required current checkpoints
  and retained event versions.
- [ ] Candidate can safely reject stale terms/generations/epochs at every
  write/commit boundary.
- [ ] Quorum-loss, leader-change, network-partition, drain-timeout, and
  ambiguous-sink drills passed on a representative isolated cluster.
- [ ] Telemetry/alerts for quorum, leader changes, lease margin, version
  distribution, assignment churn, lag, and SLO regression are active.
- [ ] Capacity can tolerate the planned unavailable nodes plus recovery
  headroom. If not, use a maintenance outage plan, not a rolling-availability
  claim.
- [ ] Abort criteria, rollback/forward-fix, operator contact, and
  out-of-band communication are confirmed.

## Phase 1: Expand protocol compatibility

Use an expand/contract sequence whenever possible:

1. Deploy a release that can read both old and new compatible representations
   without changing authoritative semantics.
2. Keep existing nodes and producers on the old representation; verify the
   expanded release reports compatible versions and correct quorum state.
3. If coordination-store schema changes are needed, use the provider's
   quorum-safe, backward-compatible migration mechanism. Do not let each node
   independently mutate shared schema at startup.
4. Confirm every supported old/new participant combination with contract
   tests before proceeding.

Do not activate a new leader, plan generation, or writer with protocol
semantics that old participants can misinterpret.

## Phase 2: Upgrade control-plane participants

1. Confirm the current leader, quorum membership, term/epoch high-water mark,
   and healthy follower replication.
2. Upgrade one non-leader participant at a time. Keep the old leader available
   until a compatible majority has been verified, unless the provider requires
   a different documented safe order.
3. After each node returns, verify authenticated registration, generation,
   protocol compatibility, replicated log/state catch-up, and quorum health.
4. If leadership transfer is required, use the consensus provider's supported
   transfer procedure. Verify the new leader has current committed state and
   that the old leader is fenced from mutations.
5. Never take another participant offline until the required quorum and
   failure margin are restored. Stop if the quorum provider reports degraded,
   divergent, or uncertain state.
6. Upgrade remaining participants one at a time, observing leader changes,
   epoch monotonicity, rejected stale writes, and API behavior.

Control-plane replicas must not simultaneously run an unreviewed protocol
contract change. A rolling binary upgrade is allowed only inside the approved
mixed-version compatibility window.

## Phase 3: Upgrade data-plane workers

1. Select a canary role and bounded partition set. Confirm remaining nodes have
   capacity and that partitions can be restored within the drain budget.
2. Stop new ownership for the selected old worker; drain in-flight work and
   commit only contiguous durable progress.
3. Fence its assignment epoch at shared state/checkpoint/broker/sink boundaries.
4. Start the target worker with a new process generation, authenticate it,
   verify dependencies, and restore a compatible checkpoint.
5. Assign a strictly newer epoch, require acknowledgement for the exact plan
   generation and partition set, then resume only the canary work.
6. Observe processing correctness, lag, end-to-end latency, retries,
   duplicates, sink outcomes, state/checkpoint errors, resource headroom, and
   rebalance time throughout the bake window.
7. Expand by the pre-approved bounded batch size. Re-evaluate capacity,
   quorum, and guardrails after every batch.
8. Repeat independently for ingest, compute, and egress roles where their
   compatibility or state behavior differs.

Do not run old and new versions as simultaneous writers to the same partition
unless a tested protocol explicitly provides a single-writer handoff. If the
source cannot pause or defer acknowledgement, use its documented exception
path and account for its loss/duplicate window.

## Phase 4: State, schema, and routing migration

Prefer separate expand and contract releases:

1. Introduce new readers/state fields/topic lanes while retaining old forms.
2. Verify readers can process both forms, including replay from the oldest
   retained event/checkpoint that must remain recoverable.
3. Switch producers or assignments in bounded batches with stable event and
   idempotency keys.
4. Verify no old writers remain and all required checkpoints, output intents,
   and sink deduplication records have advanced beyond the old format.
5. Take and verify a recovery snapshot before destructive cleanup.
6. Remove old fields/topics/state only in a later approved release after the
   compatibility/retention window expires.

Partition-key or partition-count changes require a versioned mapping plan,
state migration, ordering analysis, and explicit checkpoint boundary. Do not
assume a hash change preserves ownership or key order.

## Stop and rollback decision

Stop further rollout immediately on:

- quorum loss, conflicting control histories, or term/epoch regression;
- accepted stale-owner writes or two valid owners for one partition;
- checkpoint advancement beyond durable state/output or unexplained offset
  gaps;
- unresolvable ambiguous sink outcomes, data-loss/duplicate budget breach;
- sustained SLO regression, runaway lag, or exhausted resource headroom;
- mixed-version incompatibility or inability to restore prior state.

When safe:

1. Freeze rollout, automated scaling, and unrelated topology changes.
2. Keep quorum and current authoritative terms/epochs intact.
3. Roll back only the binary/configuration components proven compatible with
   current committed state and messages.
4. If schema/state conversion is irreversible, use the tested forward-fix or
   restore procedure; do not force an old binary to read incompatible state.
5. Reconcile node generations, partition owners, source positions, checkpoints,
   output intents, and sink idempotency records.
6. Resume canary partitions only after fencing and reconciliation pass; then
   expand under approval.

Never use offset reset, deletion of checkpoints, term reset, or epoch reuse as
a generic rollback mechanism.

## Completion criteria

- [ ] All intended control participants are on approved versions and quorum
  replication is healthy.
- [ ] All active workers report target versions and exact current plan/epoch.
- [ ] No stale generations or conflicting owners remain.
- [ ] State/checkpoint/schema and retained-message replay are compatible.
- [ ] Input/output positions show no unexplained gaps; sink outcomes reconcile.
- [ ] SLO, lag, duplicate/loss budgets, and capacity headroom are within
  approved objectives for the bake window.
- [ ] Old protocol/schema support is removed only if the contract window and
  recovery-retention window have both expired.
- [ ] Release record, evidence, exceptions, and follow-up owners are complete.

Record the outcome in
[`../deployment/release-readiness.template.md`](../deployment/release-readiness.template.md)
and use
[`recovery-drill-record.template.md`](recovery-drill-record.template.md) for
the rehearsal evidence.
