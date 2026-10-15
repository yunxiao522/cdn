# Tenant Isolation, Fair Scheduling, and Admission Control

This document defines an optional multi-tenant resource-governance model for
the distributed streaming design. It is not implemented or enforced by the
repository. The example policy in
[`../config/examples/tenant-policy.example.yaml`](../config/examples/tenant-policy.example.yaml)
is illustrative only; all limits and weights require workload measurement and
owner approval.

## Identity and trust boundary

- Resolve tenant identity from authenticated workload identity and
  control-plane configuration. Do not trust a tenant ID supplied only in the
  event payload, topic name, HTTP header, or client-selected pipeline field.
- Bind each pipeline/source/sink to an authorized tenant at registration or
  deployment time. Authorize resource creation, quota changes, replay, and
  dead-letter access separately.
- Carry the resolved tenant/pipeline identity as protected processing metadata
  across bus, state, checkpoint, retry, and sink boundaries. Preserve it on
  replay and audit any administrative reassignment.
- Reject or quarantine unauthenticated/unknown tenant traffic before durable
  acceptance. Never silently map unknown identity to a privileged default.
- Avoid raw tenant IDs as unbounded telemetry dimensions. Use approved bounded
  identifiers or aggregation buckets and enforce access controls on
  tenant-specific dashboards and logs.

## Resource isolation model

Set independent limits by tenant and resource type:

| Resource | Example bound | Purpose |
| --- | --- | --- |
| Ingress rate | Events/s and bytes/s with burst allowance | Prevent one producer from consuming shared ingress capacity |
| Event size | Maximum bytes and parse/validation cost | Bound memory, CPU, and serialization overhead |
| In-flight work | Events and bytes, per tenant and worker | Bound queues and working-set memory |
| Concurrency | Active requests, partitions, or processing slots | Prevent noisy-neighbor saturation and uncontrolled fan-out |
| Retry/dead-letter | Backlog bytes, age, and event count | Prevent failures from consuming unbounded capacity |
| State | Keys/bytes and operation rate where supported | Protect shared state latency and storage |
| Egress | Requests/s, bytes/s, and destination concurrency | Respect sink quotas and isolate slow destinations |

Resource accounting must include buffering and retries, not only successfully
processed events. A count-only limit is insufficient when event sizes vary.
Validate aggregate tenant maxima against the physical capacity of each shared
dependency; per-tenant limits that sum above cluster capacity do not provide
isolation.

Separate critical shared control-plane capacity from tenant data-plane budgets.
Data-plane overload must not exhaust resources required for membership,
fencing, health reporting, or safe drain.

## Admission and overload

1. Authenticate and authorize the tenant/pipeline before accepting the
   workload.
2. Validate envelope size, schema, allowed source/sink, and configured quota.
3. Apply rate limits to both events and bytes, with a bounded burst allowance.
   Token-bucket or equivalent admission algorithms are acceptable only when
   their distributed behavior and state-loss semantics are documented.
4. Check shared dependency health and remaining bounded capacity. Do not accept
   work durably if it cannot be retained within the approved outage/replay
   budget.
5. On a soft limit, apply source-native backpressure or pause consumption.
   Return/retry with an explicit bounded delay when the protocol supports it.
6. On a hard limit or unauthorized request, reject before durable acceptance
   with a stable error class and telemetry; do not acknowledge the source
   record as successful.
7. Define behavior during quota-store or policy-service outage. Do not fail
   open to unlimited acceptance; use only a bounded, explicitly approved local
   policy cache, otherwise fail closed for new admission.

Avoid retry storms: use retry budgets, jitter, and per-tenant circuit breaking.
When the downstream sink is slow, pause that tenant's egress lane or isolate
its bounded output backlog rather than allowing it to consume every worker's
memory.

## Fair scheduling

Choose a scheduling algorithm that matches the fairness objective and supports
the required ordering semantics. Weighted deficit round robin or weighted fair
queuing can share worker slots across eligible tenants, but neither guarantees
equal latency under unequal workloads or partition constraints.

- Schedule only among work that is ready, authorized, and within its tenant
  budget.
- Bound the maximum consecutive work and bytes per tenant before yielding to
  another eligible tenant.
- Keep per-partition ordering: fair scheduling may interleave independent
  partitions, but must not reorder events within an ordered partition.
- Define weights and priority classes with owners and audit changes. Priority
  must not bypass authentication, fencing, memory bounds, or safe commit
  ordering.
- Prevent starvation with a measured maximum wait/age objective and aging or
  equivalent policy.
- Handle a tenant with one hot partition explicitly; adding replicas cannot
  parallelize that ordered partition without a contract change.
- Specify where fairness is enforced: ingress, compute, state-store access, and
  egress may each require independent scheduling. Fair ingress alone does not
  prevent one tenant from monopolizing shared state or sink capacity.

## Failure and policy changes

- Quota changes are versioned, authorized, audited, and applied consistently
  across active workers. Define propagation delay and behavior when workers
  hold different policy versions.
- Reducing a limit below current use must be gradual: stop new admission first,
  drain or expire existing work according to policy, and never discard
  acknowledged data to meet a quota.
- On worker reassignment, restore per-tenant accounting or reconcile it from a
  durable source. Resetting counters at restart must not create an unlimited
  burst window.
- During dependency degradation or incident recovery, preserve reserved
  capacity for control and recovery work; freeze autoscaling and tenant
  policy changes unless the incident commander approves otherwise.
- Tenant suspension must stop new intake and fence its active processing
  leases, while retaining accepted data and audit history under the configured
  retention policy.

## Metrics and acceptance tests

Measure admitted/rejected events and bytes, throttling duration, per-tenant
queue age, in-flight bytes, retry/dead-letter backlog, processing share,
starvation wait, and quota-policy version. Keep tenant labels bounded and
access-controlled as described in
[`observability.md`](observability.md).

Before enabling multi-tenancy, test:

- one tenant exceeding event and byte limits while another remains within
  limits;
- a single tenant saturating a hot partition, shared state, or sink;
- quota-service outage and stale local policy cache;
- retry/dead-letter buildup and recovery without cross-tenant starvation;
- fairness across different event sizes and sustained/bursty workloads;
- tenant suspension, policy update, worker restart, and partition rebalance;
- that no unauthorized tenant can read, replay, or affect another tenant's
  events, state, output, logs, or telemetry.

Pass criteria must quantify isolation, allowed starvation latency, bounded
memory/backlog, throughput, and impact on shared control-plane safety. Record
the approved limits in the production capacity and service-objective worksheets;
do not infer them from the sample file.
