# Distributed Release, Compatibility, and Rollback Plan

This is a release design template, not an implemented deployment pipeline.
Select an orchestrator and define release tooling before using these steps.

## Compatibility requirements

- Version node/control API and event schemas independently where necessary.
- Prefer additive API/schema changes first; readers must tolerate unknown
  optional fields and old supported versions.
- Do not remove or reinterpret fields until all producers, consumers, retained
  messages, checkpoints, and restore paths have moved past the old version.
- Store processing and schema versions with checkpoints/commit metadata so
  replay behavior is explicit.
- Define a supported compatibility window and minimum/maximum control protocol
  versions before allowing mixed-version membership.
- Treat assignment epoch and fencing rules as protocol-critical; incompatible
  fencing behavior requires a coordinated upgrade, not a rolling mixed mode.

## Progressive rollout

1. Build immutable, signed artifacts and produce an auditable software/config
   manifest. Scan dependencies and verify provenance before deployment.
2. Validate configuration and schemas; run unit, contract, integration,
   performance, security, and recovery tests against the candidate.
3. Deploy to an isolated test cluster and run the relevant failure drills.
4. Deploy control-plane changes to a canary/control member only when protocol
   compatibility is established; preserve quorum during replacement.
5. Deploy one canary worker per role or a small isolated partition subset.
   Observe health, assignment churn, lag, error rates, and p99 latency for a
   predefined bake window.
6. Expand by bounded node/partition batches. Drain and checkpoint before
   replacing workers; do not overlap writers without enforced fencing.
7. Stop rollout automatically when a declared guardrail fails; page the owner
   and preserve evidence.

## Rollback

- Define rollback triggers before rollout: readiness failure, repeated
  rebalance, stale-epoch rejection spikes, error/duplicate/loss budget breach,
  SLO regression, or dependency instability.
- Halt further batches and freeze autoscaling/rebalancing if that is safer.
- Roll back only to a version compatible with current schemas, checkpoints,
  and control epochs. Never roll back epoch numbers or reuse fencing tokens.
- If data format or state migration is irreversible, use a tested forward-fix
  or restore plan rather than an unsafe binary rollback.
- Verify partition ownership, committed positions, sink idempotency, and
  cluster health after rollback; record the incident and remediation.

## Release record

For each release, retain artifact digest/signature, source revision, config and
schema versions, compatibility matrix, canary/bake results, rollout batches,
rollback decision, operator approvals, and post-deployment SLO measurements.
