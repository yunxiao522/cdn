# Disaster Recovery Runbook (Template)

This runbook is intentionally provider-neutral and incomplete until operators
fill in approved contacts, service endpoints, credentials access, and RPO/RTO.
It is not an executable recovery command list.

## Incident record

- Incident ID / bridge: `TBD`
- Incident commander / recovery lead: `TBD`
- Affected cluster, region, failure domains: `TBD`
- Start time (UTC) / detection source: `TBD`
- Suspected impact and affected sources, topics, partitions, sinks: `TBD`
- Approved RPO / RTO for this service: `TBD`
- Change/fault that preceded incident, if known: `TBD`

## Safety gates

1. Declare incident, assign incident commander, and use the approved
   out-of-band communications channel.
2. Stop destructive automation, retention/compaction jobs, and nonessential
   deployments. Freeze autoscaling if it may worsen a rebalance or dependency
   incident.
3. Determine whether any old owners can still write. Fence affected node
   generations/assignment epochs before granting replacement ownership.
4. Do not delete, overwrite, or restore over shared state until its current
   contents and available backup/snapshot manifests are recorded.
5. If authority, fencing, or data integrity cannot be established, pause
   affected partitions. Prefer bounded unavailability to split-brain writes.

## Recovery procedure

### A. Assess and contain

- Identify failed services and failure domains from control-plane, broker,
  state-store, checkpoint-store, network, and telemetry health.
- Compare last committed offsets/checkpoints with current broker positions.
- List affected partition IDs, owners, generations, and epochs.
- Record uncertain sink outcomes and isolate consumers that could duplicate
  non-idempotent effects.
- Decide whether this is a failover, in-place repair, or restore-from-backup
  event; obtain required approval.

### B. Restore authority and dependencies

- Restore control-plane quorum and authoritative membership/epoch state first.
- Restore message-bus availability and verify retained replay range.
- Restore shared state and checkpoint services from a consistent snapshot and
  manifest; verify checksums and schema compatibility.
- Confirm identity, secret access, network policy, and observability are
  available without exposing secret values.

### C. Reassign and replay

- Fence former owners and advance assignment epochs using the approved
  coordination mechanism.
- Select the last verified committed checkpoint for each affected partition.
- Restore state corresponding to that checkpoint and replay subsequent durable
  events in order.
- Use original idempotency keys; reconcile ambiguous sink writes before
  re-delivery.
- Re-drive dead-letter events only after root cause is fixed and approval is
  recorded.

### D. Validate and resume

- Resume a small canary partition set first, then expand in bounded batches.
- Verify no stale-owner writes, offset gaps, unexplained duplicates, schema
  incompatibility, or rising error/dead-letter counts.
- Confirm input/output lag and end-to-end latency recover to their approved
  objectives for the defined observation window.
- Resume automated scaling and retention only after the incident commander
  approves.
- Record remaining data gaps, customer impact, and follow-up actions.

## Exit criteria

Recovery is complete only when all criteria are explicitly checked:

- [ ] Control-plane quorum and authoritative epochs are healthy.
- [ ] Every active partition has exactly one fenced, acknowledged owner.
- [ ] Shared state/checkpoints are consistent and within approved RPO.
- [ ] Source/broker offsets have no unexplained gaps.
- [ ] Sink outcomes and duplicate handling are reconciled.
- [ ] SLOs, lag, and error rates are within target for `TBD` observation window.
- [ ] Backups, retention, alerting, and scaling automation are safe to resume.
- [ ] Incident timeline and evidence are preserved; post-incident review scheduled.

## Recovery drill worksheet

Copy [`recovery-drill-record.template.md`](recovery-drill-record.template.md)
for every exercise. Never run destructive recovery steps against production
without explicit approval, tested backups, and a bounded rollback plan.
