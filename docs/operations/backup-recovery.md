# Distributed Backup, Recovery, and Retention Plan

This is an operational design outline, not an implemented backup utility or
recovery guarantee. The project currently has no runtime capable of creating or
restoring backups.
For incident-time restore steps and an exercise record, use
[`disaster-recovery-runbook.md`](disaster-recovery-runbook.md) and
[`recovery-drill-record.template.md`](recovery-drill-record.template.md).

## Storage responsibilities

- **Message bus:** replicated event log and retained source offsets. Availability
  and replication guarantees depend on the selected broker.
- **State store:** authoritative keyed state and partition ownership metadata.
- **Checkpoint store:** coordinated per-partition progress and recovery
  manifests.
- **Object/archive storage:** snapshots, retained cold data, and quarantine
  payloads where required by policy.
- `config/retention/` is reserved for versioned retention policy templates.

These are external distributed services, not directories on one application
node. Backups must be stored in a separately managed failure domain; this
repository configures neither storage nor credentials.

## Planned backup procedure

1. Record cluster, partition ownership, committed offsets, schema versions, and
   deployment/configuration version.
2. Request a consistent snapshot from the selected state/checkpoint services
   coordinated with message-bus offsets.
3. Replicate snapshot data and required retained message ranges to a separately
   managed backup failure domain, with encryption and integrity verification.
4. Record a manifest containing cluster identity, partition/offset ranges,
   snapshot IDs, schema versions, creation time, and checksums.
5. Confirm the backup can be read and periodically restore it into an isolated
   test cluster.

## Planned recovery procedure

1. Fence affected workers and pause or isolate source consumption for impacted
   partitions.
2. Restore the most recent valid shared-state snapshot and verify its manifest.
3. Restore/replay retained message ranges after each partition's checkpoint.
4. Reconcile partition ownership, source offsets, and sink commit records before
   resuming those partitions.
5. Verify sink idempotency/deduplication to avoid duplicating already delivered
   events; resume partitions gradually while monitoring lag and errors.

## Retention and deletion

Define retention for broker topics, snapshots, checkpoints, cold/archive data,
and quarantine. Preserve enough message history to meet recovery objectives.
Only expire state or message ranges after verified snapshots and required
backups are complete. Apply retention through the selected platform with an
audit trail, not manual deletion of node-local directories.

Set recovery point/time objectives (RPO/RTO), replication factors, failure
domains, encryption, access controls, and restore-test frequency from the
service requirements. Validate them through failure-injection and restore
exercises before production use.
