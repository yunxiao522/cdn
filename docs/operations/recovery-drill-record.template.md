# Recovery Drill Record

Copy this file for each approved exercise and fill in all fields.

## Scope and approval

- Drill ID / date (UTC): `TBD`
- Environment and cluster version: `TBD`
- Scenario (node, zone, broker, state, control plane, site): `TBD`
- Exercise owner / approver / observers: `TBD`
- Fault injection method and stop procedure: `TBD`
- Affected topics and partitions: `TBD`
- Expected delivery semantics: `TBD`
- Target RPO / RTO / SLO: `TBD`

## Baseline

- Last committed checkpoint/offset per affected partition: `TBD`
- Broker end offsets and retained replay window: `TBD`
- Assignment epochs and owners: `TBD`
- Heartbeat/lease policy version and observed lease safety margin: `TBD`
- Pipeline plan generation and scheduler policy version: `TBD`
- Barrier ID and participant partitions, if applicable: `TBD`
- State/checkpoint snapshot manifest and checksum: `TBD`
- Sink idempotency/deduplication state: `TBD`
- Baseline lag, p99 latency, errors, and resource use: `TBD`

## Execution timeline

| Time (UTC) | Action / observation | Owner | Evidence reference |
| --- | --- | --- | --- |
| `TBD` | Start drill / inject fault | `TBD` | `TBD` |
| `TBD` | Detect and declare degraded state | `TBD` | `TBD` |
| `TBD` | Fence/reassign/restore/replay action | `TBD` | `TBD` |
| `TBD` | Validate and resume | `TBD` | `TBD` |
| `TBD` | End drill and restore normal operations | `TBD` | `TBD` |

## Results

- Observed RPO / RTO: `TBD`
- Lost, duplicated, delayed, or quarantined events: `TBD`
- Maximum lag and time to return to objective: `TBD`
- Stale epoch writes rejected: `TBD`
- Stale generation/plan messages rejected: `TBD`
- Assignments moved / rebalance churn / scheduler decision reason: `TBD`
- Barrier completion/abort result and wait duration, if applicable: `TBD`
- Sink ambiguous outcomes reconciled: `TBD`
- Exit criteria passed/failed: `TBD`
- Safety stop invoked? Why? `TBD`
- Unexpected behavior / evidence: `TBD`

## Follow-up

| Finding / action | Severity | Owner | Due date | Retest reference |
| --- | --- | --- | --- | --- |
| `TBD` | `TBD` | `TBD` | `TBD` | `TBD` |

Sanitize logs and evidence before sharing; exclude credentials, tokens,
private data, and unrestricted payload samples.
