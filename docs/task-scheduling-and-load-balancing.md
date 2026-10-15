# Multi-Node Task Scheduling and Load Balancing

This is a technology-neutral scheduler design, not an implemented scheduler.
The control plane makes coarse placement and ownership decisions; workers
execute events on the data plane. The scheduler must optimize only among
placements that preserve partition ownership, tenant policy, durability, and
failure-domain constraints.

## Work and placement model

Represent scheduled work as:

- pipeline/stage ID and plan generation;
- source/topic partition or other independently checkpointable work unit;
- required role/capabilities and protocol/schema compatibility;
- state locality and restore/checkpoint reference;
- resource estimate (CPU, memory, network, state I/O) and maximum in-flight
  work;
- tenant identity, quota/weight, and priority policy;
- ordering/affinity constraints and eligible failure domains;
- current owner, assignment epoch, lag/age, and last durable progress.

Do not schedule individual events through the control plane. Use partitions,
shards, or bounded task groups to amortize coordination. The chosen unit must
not split ordered state or checkpoint semantics.

## Scheduling objectives and hard constraints

Apply hard constraints before optimization:

1. Authenticated, healthy node with matching role/capability and compatible
   protocol, schema, and plan generation.
2. Available resource budget under configured limits, including recovery and
   failover headroom.
3. Exactly one active owner for each ordered work unit; old owners fenced
   before new owners can commit.
4. Tenant quota, authorization, and allowed source/sink bindings.
5. Failure-domain and minimum-surviving-capacity requirements.

Among eligible placements, balance these objectives according to reviewed
weights:

- minimize sustained queue age/lag and avoid hot-node saturation;
- preserve state/cache locality and reduce restore/checkpoint cost;
- reduce unnecessary partition movement and rebalance churn;
- spread replicas and critical partitions across failure domains;
- honor tenant fair-share weights and prevent starvation;
- keep downstream broker, state-store, and sink load within safe capacity.

No single scalar “load” is sufficient. Consider role-specific signals:
source lag/age for ingest, per-partition consumer lag and processing time for
compute, output lag and sink saturation for egress, plus resource and shared
dependency health. Use fresh signals with defined missing/stale behavior.

## Assignment and rebalance algorithm contract

1. Compute a proposed placement from a consistent membership/capacity
   snapshot. Include snapshot version and plan generation in the proposal.
2. Reject candidates whose health, capacity, tenant, capability, or
   failure-domain requirements do not pass.
3. Limit assignments moved per rebalance and preserve a configured minimum
   stable period/cooldown to prevent oscillation.
4. For each moved partition, stop intake at the old owner, commit only
   contiguous durable progress, and drain in-flight work.
5. Advance the assignment epoch in the authoritative coordination store using
   conditional update. Fence the old epoch at state/checkpoint/broker/sink
   boundaries before activating the new owner.
6. Restore compatible state at the target, verify checkpoint and source
   position, then require acknowledgement for the exact plan generation and
   epoch.
7. Resume only after readiness checks pass. If preparation times out, leave the
   partition paused or restore the prior safe owner under a valid newer plan;
   never let both generations write concurrently.
8. Observe lag, latency, resource pressure, and failure signals after each
   bounded batch. Hold, roll back safely, or alert when objectives regress.

Use sticky/minimum-movement assignment when locality and state restore are
expensive, subject to balancing and failure-domain constraints. Use
least-loaded or weighted strategies only with explicit skew handling and
stable movement thresholds. A strategy name is not a correctness guarantee;
fencing and safe transition ordering are required for every strategy.

## Load balancing by work type

| Work type | Preferred balancing unit | Important constraint |
| --- | --- | --- |
| Keyed stream processing | Broker/source partition or stable key group | Preserve per-key order and state locality |
| Stateless processing | Bounded partition/task batch | Bound in-flight bytes and downstream concurrency |
| HTTP polling | Query shard, cursor range, or distributed lease | Cursor updates after durable publication; no unsafe offset pagination |
| MQTT | Broker shared-subscription delivery unit/session | Verify broker semantics and acknowledgement timing |
| Shared files | Immutable file/object claim or manifest shard | Distributed lease/fencing and durable record progress |
| Sink delivery | Output-intent partition or destination quota lane | Stable idempotency key and destination rate limits |

Never hash records independently at each stage if that breaks key affinity.
Rescaling partition count can remap keys; use an explicit migration/versioning
plan and preserve checkpoint compatibility.

## Backpressure and overload

- Pause or reduce source intake when durable downstream capacity is threatened;
  do not shift unbounded queues into worker memory.
- Isolate a slow sink or hot tenant with bounded concurrency/backlog, without
  violating required ordering or acknowledging work that is not durable.
- Do not move a bottlenecked partition repeatedly between equally saturated
  nodes. Require evidence of available capacity before reassignment.
- Freeze automatic scheduling changes during control-plane quorum loss,
  uncertain leases, dependency incidents, recovery, stale metrics, or repeated
  failed rebalances.
- If the system cannot find a safe placement, leave the task pending/paused,
  expose the reason, and alert; do not choose an ineligible node as a fallback.

## Scheduler observability

Measure eligible/ineligible nodes and bounded rejection reasons, task/partition
lag and age, node role saturation, assignment movement count, rebalance
duration, state restore time, placement skew, tenant wait time, scheduling
decision age, and stale plan/epoch rejections. Do not label metrics with
unbounded event IDs or arbitrary task text.

Every decision should be auditable with plan generation, candidate set summary,
selected owner, reason class, relevant policy version, and epoch transition.
Protect detailed topology/tenant data with access control.

## Required tests

- Balanced and skewed partitions at steady load and burst load.
- Node joining/leaving during assignment and plan-generation updates.
- Worker capacity report stale, missing, or contradictory.
- Tenant quota and priority/fairness interactions.
- Hot partition that cannot be split without changing ordering contract.
- State restore cost and rebalance limit under scale-out/scale-in.
- Sink throttling or shared-state saturation where scaling workers would
  worsen load.
- Drain timeout, coordinator restart, stale owner resumes, and fencing denial.
- Failure-domain loss while preserving required control quorum and minimum
  surviving workers.

Pass criteria include no dual writers, no uncommitted progress advancement,
bounded movement/churn, no starvation beyond the approved objective, and
measured SLO impact within the approved budget. Record thresholds in
[`performance-capacity.md`](performance-capacity.md) and
[`multi-instance-operations.md`](multi-instance-operations.md).
