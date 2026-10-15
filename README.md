# Distributed Real-Time Data Runtime

Language-agnostic structure for a multi-node, multi-source streaming system.
This repository contains architecture and configuration templates only; it
does not contain an executable cluster, connectors, control plane, or worker
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
│   │   └── topics.example.yaml
│   ├── retention/                    # Data retention policy templates
│   └── security/                     # Policy templates; never live secrets
├── deployments/
│   └── topologies/
│       ├── cluster.example.yaml      # Technology-neutral topology intent
│       └── production-capacity.template.yaml
├── docs/
│   ├── deployment/
│   │   ├── dependencies.md
│   │   └── release-rollout.md
│   ├── cluster-protocol.md
│   ├── consistency-delivery.md
│   ├── data-contracts-and-topics.md
│   ├── operations/
│   │   ├── backup-recovery.md
│   │   ├── failure-drills.md
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
│   │   ├── membership/
│   │   ├── partition-assignment/
│   │   └── scheduling/
│   ├── data-plane/
│   │   ├── compute/
│   │   ├── ingest/
│   │   └── egress/
│   └── platform/
│       ├── checkpointing/
│       ├── coordination/
│       ├── health/
│       ├── messaging/
│       ├── observability/
│       ├── security/
│       └── state/
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

`HTTP / Kafka / MQTT / shared files` → `source connectors` → `durable shared
message bus` → `partitioned worker groups` → `sink connectors`

The control plane manages node membership, health, and partition assignment.
The data plane performs ingestion, computation, and delivery. A shared durable
message bus and shared state/checkpoint services provide cross-node recovery
boundaries. Their products and delivery semantics remain undecided.

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

## Scope and constraints

All configuration and topology files are illustrative, not deployable
manifests. There is currently no command that starts a cluster. A production
deployment must select the orchestration platform, broker, shared state and
checkpoint stores, identity provider, secrets manager, and storage providers.

This structure targets distributed low-latency stream processing on general
purpose infrastructure, not deterministic hard-real-time behavior. Keep
credentials in an approved secrets manager; never commit credentials,
certificates, private keys, or production endpoints.
