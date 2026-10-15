# Distributed Real-Time Data Runtime

Language-agnostic architecture and configuration templates for centralized
and distributed multi-source streaming deployments. This repository does not
contain an executable cluster, connectors, control plane, or worker
implementation. It does not claim hard-real-time guarantees.

## Planned directory layout

```text
.
├── benchmarks/
│   └── scenarios/                    # Reproducible workload plans
├── config/
│   ├── examples/
│   │   ├── pipeline.example.yaml
│   │   ├── runtime.example.yaml
│   │   ├── sources.example.yaml
│   │   ├── tenant-policy.example.yaml
│   │   └── topics.example.yaml
│   ├── retention/                    # Data retention policy templates
│   └── security/                     # Policy templates; never live secrets
├── deployments/
│   └── topologies/
│       ├── cluster.example.yaml      # Technology-neutral topology intent
│       ├── full-node.example.yaml    # Full-node control + data capability intent
│       └── production-capacity.template.yaml
├── docs/
│   ├── deployment/
│   │   ├── dependencies.md
│   │   ├── cluster-bootstrap-runbook.md
│   │   ├── release-readiness.template.md
│   │   └── release-rollout.md
│   ├── cross-node-coordination.md
│   ├── node-communication.md
│   ├── task-scheduling-and-load-balancing.md
│   ├── cluster-protocol.md
│   ├── consistency-delivery.md
│   ├── data-contracts-and-topics.md
│   ├── architecture-modes.md
│   ├── full-node-cluster.md
│   ├── multi-instance-operations.md
│   ├── observability.md
│   ├── processing-lifecycle.md
│   ├── source-adapters.md
│   ├── sink-adapters.md
│   ├── tenant-isolation-and-admission.md
│   ├── operations/
│   │   ├── backup-recovery.md
│   │   ├── failure-drills.md
│   │   ├── full-node-upgrade-runbook.md
│   │   ├── disaster-recovery-runbook.md
│   │   ├── recovery-drill-record.template.md
│   │   ├── security.md
│   │   └── service-objectives.md
│   ├── performance-capacity.md
│   └── runtime-architecture.md
├── interfaces/
│   └── http/
│       └── openapi.yaml              # Planned control-plane and health API
├── monitoring/
│   ├── alerts/
│   └── dashboards/
├── scripts/                          # Future cluster tooling
├── src/
│   ├── node-agent/                   # Future per-node lifecycle/health agent
│   ├── connectors/
│   │   ├── sinks/
│   │   └── sources/
│   │       ├── file/
│   │       ├── http/
│   │       ├── kafka/
│   │       └── mqtt/
│   ├── contracts/
│   │   ├── migrations/
│   │   └── schemas/
│   │       └── v1/
│   │           └── event.schema.json
│   ├── control-plane/
│   │   ├── api/
│   │   ├── cluster-coordination/
│   │   ├── membership/
│   │   ├── partition-assignment/
│   │   └── scheduling/
│   ├── data-plane/
│   │   ├── compute/
│   │   ├── coordination/
│   │   ├── ingest/
│   │   └── egress/
│   └── platform/
│       ├── checkpointing/
│       ├── coordination/
│       ├── health/
│       ├── messaging/
│       ├── observability/
│       ├── security/
│       ├── state/
│       └── transport/
└── tests/
    ├── contract/
    ├── failure/
    ├── integration/
    ├── performance/
    ├── security/
    └── unit/
```

Empty implementation directories are retained by `.gitkeep`. Durable data,
logs, checkpoints, queues, and secrets are owned by external cluster services
or deployment-managed volumes; they do not belong in this source tree.

## Cluster data flow

`HTTP / Kafka / MQTT / shared files` → `multi-node source connectors` →
`durable shared message bus` → `partitioned worker groups` → `sink connectors`

The target shape is a logically centralized, quorum-backed control plane
managing a multi-node data plane. The control plane manages membership, health,
and partition assignment; it does not proxy event payloads. The data plane
performs ingestion, computation, and delivery across nodes. A shared durable
message bus and shared state/checkpoint services provide cross-node recovery
boundaries. Their products and delivery semantics remain undecided.
Cluster-level assignment and pipeline-level cross-node coordination are
separate responsibilities; see
[cross-node-coordination.md](docs/cross-node-coordination.md).
The full-node profile puts control participation and data-processing capability
on each node while preserving quorum authority and single-owner partition
execution; see [full-node-cluster.md](docs/full-node-cluster.md) and
[full-node.example.yaml](deployments/topologies/full-node.example.yaml).
The full-node rolling upgrade and protocol/state migration procedure is
documented in
[full-node-upgrade-runbook.md](docs/operations/full-node-upgrade-runbook.md).
Provider-neutral multi-node start, canary, abort, and safe shutdown order is
documented in
[cluster-bootstrap-runbook.md](docs/deployment/cluster-bootstrap-runbook.md).

See [docs/runtime-architecture.md](docs/runtime-architecture.md) for node roles,
failure recovery, scaling flows, deployment order, and consistency boundaries.
Membership, partition-assignment, and cross-service consistency semantics are
specified in [cluster-protocol.md](docs/cluster-protocol.md).
Network-partition safety, fencing, delivery, and replay semantics are specified
in [consistency-delivery.md](docs/consistency-delivery.md). The production
capacity worksheet is [production-capacity.template.yaml](deployments/topologies/production-capacity.template.yaml).
The protocol security review checklist is included there. Failure drills and
acceptance criteria are in [failure-drills.md](docs/operations/failure-drills.md);
horizontal scaling policy inputs and decision sequence are in
[performance-capacity.md](docs/performance-capacity.md).
Service objectives, RPO/RTO, and alert templates are in
[service-objectives.md](docs/operations/service-objectives.md). API identity and
credential rotation are covered by [security.md](docs/operations/security.md);
version compatibility, canary rollout, and rollback are covered in
[release-rollout.md](docs/deployment/release-rollout.md).
The service dependency and environment checklist is
[dependencies.md](docs/deployment/dependencies.md). The topology template is
[cluster.example.yaml](deployments/topologies/cluster.example.yaml). The planned
node registration, heartbeat, partition assignment, and health APIs are
specified in [openapi.yaml](interfaces/http/openapi.yaml). Performance targets
and capacity planning are covered in [performance-capacity.md](docs/performance-capacity.md).
The event envelope, logical topics, and partitioning rules are in
[data-contracts-and-topics.md](docs/data-contracts-and-topics.md). The dependency
graph is in [dependencies.md](docs/deployment/dependencies.md); disaster recovery
steps and the drill record are in
[disaster-recovery-runbook.md](docs/operations/disaster-recovery-runbook.md) and
[recovery-drill-record.template.md](docs/operations/recovery-drill-record.template.md).
The end-to-end source, compute, and sink commit lifecycle is specified in
[processing-lifecycle.md](docs/processing-lifecycle.md).
Source-specific cursor, offset, acknowledgement, ownership, and backpressure
requirements are in [source-adapters.md](docs/source-adapters.md).
Destination idempotency, confirmation, batching, ordering, and retry contracts
are in [sink-adapters.md](docs/sink-adapters.md).
Required metrics, structured logging, tracing, cardinality, alerting, and
telemetry safety are defined in [observability.md](docs/observability.md).
Per-change rollout gates, evidence, approvals, and rollback tracking are
captured in [release-readiness.template.md](docs/deployment/release-readiness.template.md).
Multi-instance placement, partition-limited scaling, backpressure bounds, and
throughput acceptance are covered in
[multi-instance-operations.md](docs/multi-instance-operations.md).
Tenant identity, quotas, fair scheduling, admission control, and overload
isolation are described in
[tenant-isolation-and-admission.md](docs/tenant-isolation-and-admission.md), with
an illustrative policy at
[tenant-policy.example.yaml](config/examples/tenant-policy.example.yaml).
Centralized-control multi-node topology, federated-cluster trade-offs, shared
contracts, and a safe single-node migration path are described in
[architecture-modes.md](docs/architecture-modes.md).
The control-plane cluster coordination, data-plane coordination, multi-worker
execution sequence, barriers, and failure behavior are described in
[cross-node-coordination.md](docs/cross-node-coordination.md).
Node communication, heartbeat/lease constraints, scheduling and load balancing
are described in
[node-communication.md](docs/node-communication.md) and
[task-scheduling-and-load-balancing.md](docs/task-scheduling-and-load-balancing.md).
Multi-node fault scenarios and their acceptance checks are in
[failure-drills.md](docs/operations/failure-drills.md).

## Scope and constraints

All configuration and topology files are illustrative, not deployable
manifests. There is currently no command that starts a cluster. The documents
cover centralized and distributed deployment modes, but neither is
implemented. A production deployment must select the orchestration platform,
broker, shared state and checkpoint stores, identity provider, secrets manager,
and storage providers.

This structure targets distributed low-latency stream processing on general
purpose infrastructure, not deterministic hard-real-time behavior. Keep
credentials in an approved secrets manager; never commit credentials,
certificates, private keys, or production endpoints.
