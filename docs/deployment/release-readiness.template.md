# Release Readiness Record

Copy this template for each software, configuration, schema, topology, or
dependency change. A `TBD` field is not an approval or a passing result.
Release steps are planning guidance only; this repository has no deployment
pipeline.

## Change identity and ownership

- Release/change ID: `TBD`
- Source revision and immutable artifact digest: `TBD`
- Environment and affected clusters: `TBD`
- Change owner / operator / approver: `TBD`
- Planned window and communication channel: `TBD`
- Change type: `software / config / schema / topology / dependency`
- Related incident, issue, or change record: `TBD`

## Impact and compatibility

- Affected roles, pipelines, sources, sinks, topics, and partitions: `TBD`
- Current and target software/config/schema versions: `TBD`
- Compatibility matrix for mixed-version workers and control plane: `TBD`
- Retained-message and checkpoint replay compatibility verified: `TBD`
- State/schema migration or dual-read/write plan: `TBD`
- Partition-count, key-routing, ordering, or assignment impact: `TBD`
- External dependency, identity, permission, or network-policy changes: `TBD`
- Data classification and secret/logging review completed: `TBD`

## Pre-deployment gates

- [ ] Configuration and contract validation passed; diff reviewed by owner.
- [ ] Unit, contract, integration, and relevant security checks passed.
- [ ] Performance result meets the declared objective or has an approved
  exception.
- [ ] Failure/recovery tests cover the changed boundaries.
- [ ] Backup/snapshot and restore path verified if state or storage changes.
- [ ] Canary target, batch size, bake window, and stop authority are recorded.
- [ ] Rollback or forward-fix is tested and compatible with current state,
  schemas, offsets, and fencing epochs.
- [ ] Alerts, dashboards, telemetry redaction, and runbook links are ready.
- [ ] Required service owner and operations approvals are recorded.

Evidence references: `TBD`

## Rollout and guardrails

| Stage | Scope / batch | Start and end time (UTC) | Health evidence | Decision / approver |
| --- | --- | --- | --- | --- |
| Canary | `TBD` | `TBD` | `TBD` | `TBD` |
| Batch 1 | `TBD` | `TBD` | `TBD` | `TBD` |
| Batch 2 | `TBD` | `TBD` | `TBD` | `TBD` |
| Complete | `TBD` | `TBD` | `TBD` | `TBD` |

Record thresholds and actual values for:

- readiness and dependency health: `TBD`;
- assignment churn, lease margin, and stale-epoch rejections: `TBD`;
- input/output lag and oldest-event age: `TBD`;
- error, retry, duplicate, loss, and dead-letter rate: `TBD`;
- end-to-end latency percentiles and resource headroom: `TBD`.

Stop rollout if any safety invariant fails or an approved guardrail is crossed.
Do not continue merely because the canary process is alive.

## Rollback / forward-fix record

- Decision and timestamp (UTC): `TBD`
- Trigger and evidence: `TBD`
- Selected action and reason: `TBD`
- State/schema/offset compatibility checked by: `TBD`
- Ownership fenced and current epochs preserved: `TBD`
- Sink outcomes and replay/idempotency checked: `TBD`
- Validation results and remaining impact: `TBD`

## Completion

- Final software/config/schema versions: `TBD`
- SLO and recovery measurements after bake window: `TBD`
- Outstanding risks, exceptions, and expiration dates: `TBD`
- Follow-up action / owner / due date: `TBD`
- Record location and retention class: `TBD`

Do not include secret values, raw sensitive payloads, or unrestricted customer
identifiers in this record or its evidence links.
