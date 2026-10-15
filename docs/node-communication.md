# Multi-Node Communication, Heartbeats, and Leases

This document defines provider-neutral communication requirements for the
centralized control plane and multi-node data plane. It is a design contract,
not an implemented protocol or a selection of transport technology.

## Communication planes

Keep control and event-data paths distinct:

| Path | Direction | Purpose | Safety requirements |
| --- | --- | --- | --- |
| Control API | Node/agent ↔ control plane | Register, heartbeat, fetch assignment/plan, acknowledge, drain | Authenticated identity, authorization, TLS, bounded request size/rate, idempotent mutations |
| Coordination store | Control-plane replicas ↔ quorum-backed store | Membership, lease/epoch, plan and assignment authority | Strong conditional updates/CAS, quorum, durable audit, stale-write rejection |
| Event data | Ingest/compute/egress ↔ broker/state/checkpoint/sinks | Events, keyed state, durable progress, outputs | Backpressure, per-partition order, fencing or idempotency at commit boundaries |
| Telemetry | Nodes/workers → telemetry backend | Health, metrics, logs, traces | Bounded buffering, redaction, no credentials/payload dumps |

The control plane should issue compact plans and ownership metadata, not proxy
event payloads. Workers should use the selected shared data services directly
under their assigned identity and current epoch. A control API response is not
proof that the broker, state store, checkpoint store, or sink enforces fencing.

The future source organization is:

```text
src/
├── control-plane/
│   └── cluster-coordination/   # Membership, authoritative leases/epochs
├── data-plane/
│   └── coordination/           # Pipeline/task progress and barriers
└── platform/
    ├── coordination/           # CAS, quorum and lease primitives
    └── transport/              # Planned control/worker transport adapters
```

These directories identify responsibilities only. They do not imply that a
transport, RPC framework, security protocol, or server exists.

## Node identity and messages

Bind every control request to an authenticated principal and registered tuple:
`(cluster_id, node_id, generation, role)`. A client-supplied role, address, or
failure domain is descriptive input to validate, never authorization.

Control messages should carry:

- protocol version, cluster ID, request ID, and creation/expiry context;
- authenticated node ID, process generation, and role as bound by identity;
- monotonic sequence for ordered heartbeat or acknowledgement streams;
- plan generation and assignment epoch for assignment-scoped operations;
- bounded capability/resource summary and explicit result/error class.

Mutating requests must be idempotent or use a deduplication key. Reject
replayed, expired, stale-generation, stale-plan, and stale-epoch requests.
Bound payload sizes, list lengths, and per-node request rates. Return errors
that distinguish retryable dependency failure from permanent authorization or
protocol incompatibility without exposing protected cluster topology.

For secure channels, use the selected platform's workload identity and TLS
policy. Define certificate/credential rotation and bounded overlap. Do not
fall back to unauthenticated traffic during identity-provider or certificate
failure.

## Heartbeat, suspicion, and lease

Heartbeats are liveness/renewal evidence, not assignment authority. A missed
heartbeat makes a node suspect; it does not itself prove the node is unable to
write. The authoritative coordination service decides lease state, while
storage/commit boundaries enforce the monotonically increasing fencing epoch.

Define these deployment inputs and validate them under fault injection:

- heartbeat interval `H`;
- suspicion threshold `M` consecutive missed/late observations;
- lease duration `L` and renewal cadence;
- end-to-end network delay/jitter and process-pause budget;
- worker commit-stop safety margin `S`;
- control-plane decision and reassignment budget.

The configuration must leave room for the full detection and fencing path:

`L > renewal_delay_budget + decision_budget + fencing_propagation_budget + S`

This is a relationship to validate, not a recommended numeric value. Choose
`H`, `M`, and `L` from measured worst-case pauses/network behavior and service
objectives. Avoid very short timeouts that cause false eviction/rebalance
storms. The worker must stop lease-dependent commits before it can no longer
prove current authority; wall-clock time alone must not authorize a write.

Each heartbeat includes node generation, monotonic sequence, current
plan-generation acknowledgement, and bounded health/capacity summary. The
control plane returns the accepted sequence, lease status/expiry context, and
any plan version to observe. Do not put secrets, event payloads, or unbounded
diagnostic text in heartbeat messages.

On expiry, fence the old generation and assignment epochs before issuing
write-capable ownership to a replacement. If quorum or fencing cannot be
established, pause affected work. Existing workers may continue only while the
downstream lease/fencing contract remains valid.

## Retry, reconnect, and overload behavior

- Use bounded exponential backoff with jitter for transient connection errors;
  apply retry budgets and a maximum retry age.
- Re-register after reconnect with the same stable node ID and a new process
  generation when the process restarted. A network reconnect alone must not
  silently change generation.
- Fetch current authoritative plan/assignment state after reconnect. Do not
  resume from a stale local plan or cached token.
- If control-plane connectivity is unavailable, retain bounded telemetry and
  stop ownership mutations. Workers obey their lease safety margin; they must
  not self-elect or invent new assignments.
- If heartbeat/control traffic is rate-limited, shed nonessential telemetry
  before liveness/lease messages. Never let a telemetry storm starve lease
  renewal.
- Surface reconnect duration, missed-heartbeat count, lease margin, rejected
  generations/epochs, and backoff state as bounded telemetry.

## Required communication tests

Test delayed, dropped, duplicated, reordered, and replayed control messages;
node process restart with stale credentials; control-plane leader/quorum loss;
one-way and bidirectional network partitions; DNS/certificate rotation; slow
coordination store; and telemetry saturation while lease renewal remains
healthy. Verify stale nodes cannot write after fencing and that false suspicion
does not create overlapping valid owners.

See [`cluster-protocol.md`](cluster-protocol.md) for membership and assignment
state machines and [`consistency-delivery.md`](consistency-delivery.md) for
network-partition policy.
