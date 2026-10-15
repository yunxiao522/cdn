# Single-Node Real-Time Data Runtime

Language-agnostic project structure for a single-node pipeline that can be
configured for HTTP polling, Kafka, MQTT, and file-based input. This repository
currently contains structure and example configuration only: it has no
connectors, processing implementation, process supervisor, or runnable service.

## Directory layout

- `config/examples/`: illustrative source, pipeline, and runtime configuration.
- `data/input/`: staged incoming files.
- `data/work/`: transient processing data.
- `data/output/`: locally written results.
- `data/archive/`: retained input or output data.
- `deployment/`: process-role and deployment examples.
- `docs/`: design and operational notes.
- `logs/`: application and operational logs.
- `run/`: runtime state such as PID or lock files.
- `scripts/`: future start, stop, and maintenance entry points.
- `src/connectors/`: source adapters for HTTP, Kafka, MQTT, and files.
- `src/pipeline/`: future orchestration, processing, and routing logic.
- `src/outputs/`: future destination adapters.
- `tests/unit/` and `tests/integration/`: future test suites.

## Data flow

`HTTP / Kafka / MQTT / files` → `connectors` → `pipeline` → `outputs`

The example configurations are templates, not an active service configuration.
They disable external sources by default and use placeholder endpoints. Copy and
adapt them only after choosing an implementation language and runtime.

## Multi-process notes

`deployment/processes.example.yaml` describes process roles and scaling intent;
it is not a supervisor configuration. Increasing process counts requires an
implementation of inter-process work distribution, source ownership,
backpressure, and coordinated shutdown. In particular, multiple HTTP pollers
or MQTT clients can duplicate events unless partitioning or shared ownership
is implemented. Kafka consumer scaling is constrained by topic partitions.

Before connecting real sources, provide credentials through a secret manager
or environment variables; do not commit credentials or production endpoints.
