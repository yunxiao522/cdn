# Observability Contract

This is a technology-neutral instrumentation plan, not an implemented
telemetry pipeline. Select and test the metrics, logs, trace, storage, access,
and alerting backends before deployment. The target SLOs remain in
[`operations/service-objectives.md`](operations/service-objectives.md).

## Required signals

| Signal group | Minimum measurements | Primary use |
| --- | --- | --- |
| Source ingestion | Records fetched/published/acknowledged, fetch and publish failures, cursor/offset progress, oldest source age, backpressure duration | Detect source stalls, gaps, and ingest-lane outages |
| Message lanes | Publish/consume rate and bytes, queue depth, oldest record age, consumer lag, retry/dead-letter rate, retention headroom | Distinguish insufficient capacity from downstream blockage |
| Compute | Processed records, processing duration, in-flight count, state read/write latency and errors, checkpoint age/failures | Detect slow or unsafe progress |
| Ownership/control plane | Members by role/health, assignment changes, current epoch, rebalance duration, lease-renewal margin, stale-epoch rejections | Detect split-brain risk and control-plane instability |
| Egress/sinks | Attempt/success/failure counts, destination latency, ambiguous outcomes, retries, idempotency conflicts, output-lane lag | Detect delivery delay and reconcile uncertain writes |
| Shared dependencies | Availability, request latency/errors, replication lag, capacity, backup verification age | Attribute failures and protect recovery objectives |
| Runtime resources | CPU, memory, disk, network, file descriptors, restarts, saturation | Identify resource bottlenecks and capacity exhaustion |

For each counter or histogram, specify unit, aggregation semantics, reset
behavior, and owner. Report rates and latency percentiles at the source,
compute, queue, and sink boundaries; avoid averages alone. Distinguish event
time, ingestion time, processing time, and delivery time.

## Dimensions and cardinality

Use bounded dimensions where useful: `cluster`, `environment`, `region`,
`service_role`, `pipeline`, `source_type`, `sink_type`, `topic_lane`,
`partition`, `result`, and a bounded `failure_class`. Document whether
`partition` is bounded by configuration and safe for the selected metrics
backend.

Do not use `event_id`, customer/user IDs, arbitrary URLs, exception text,
payload values, credentials, or unbounded source filenames as metric labels.
These cause cardinality growth and may expose sensitive information. For
record-level diagnosis, use access-controlled logs or trace exemplars with
short retention rather than turning identifiers into time-series dimensions.

## Structured logs

Emit machine-readable records with, as applicable:

- timestamp in UTC, severity, service/component, cluster, environment, node
  ID and node generation;
- pipeline, source/sink type, topic lane, partition and assignment epoch;
- event/output-intent ID only when needed for correlation and permitted by
  data policy; keep it out of metric labels;
- operation/stage, result, bounded failure class, retry attempt, duration;
- source position or destination receipt reference only in protected logs.

Never log credentials, authorization headers, tokens, private keys, raw
environment dumps, or unrestricted event payloads. Redact sensitive fields
before serialization, including nested error objects and third-party client
debug output. Rate-limit repetitive failure logs while preserving counters and
first/last occurrence context. Keep diagnostic detail bounded and ensure
redaction failures do not silently emit the unfiltered record.

## Distributed tracing

Propagate trace context across source adapters, broker/message headers, worker
processing, output intents, and sink calls when the source/runtime supports it.
Create spans for fetch, publish, queue wait, processing, state/checkpoint
commit, sink write, and reconciliation. Record bounded error classifications
and timing; do not attach full payloads or credentials.

Sampling must preserve errors, slow traces, and recovery/replay paths at a
reviewed rate. Do not assume a trace is a complete audit trail or a durable
delivery receipt. Keep source offset, event identity, and sink receipt in the
appropriate protected audit store according to retention policy.

## Alert behavior

- Alert on user-visible symptoms (oldest-event age, end-to-end latency,
  delivery success, lag growth) as well as the dependency cause.
- Page for lease safety-margin breaches, conflicting active ownership, rejected
  stale writes, or checkpoint progress that is inconsistent with durable
  output. These are safety signals even before an SLO breach.
- Treat missing, stale, or untrusted telemetry as unknown/degraded, never as
  zero load or healthy.
- Use separate warning/page thresholds, windows, owner, runbook, and
  maintenance handling. Deduplicate related alerts but preserve affected
  pipelines/partitions in incident context.
- Every page must link to a runbook or state the safe first action. Unsafe
  automatic remediation (such as forcing reassignment) must not be triggered
  from a single telemetry sample.

The alert placeholders in
[`operations/service-objectives.md`](operations/service-objectives.md) should
be mapped to these signals, and the recovery signals should be exercised using
[`operations/failure-drills.md`](operations/failure-drills.md).

## Retention, access, and validation

Define separate retention and access policies for metrics, traces, operational
logs, audit records, and quarantined payloads. Apply access control and
encryption in transit/at rest; sanitize telemetry before export to external
providers. Ensure node-local buffering is bounded and has explicit behavior
when the telemetry backend is unavailable.

Before rollout, test dashboards and alerts with known success, failure,
backpressure, replay, and stale-epoch scenarios. Confirm alert delivery,
redaction, metric cardinality, telemetry loss behavior, and runbook links.
