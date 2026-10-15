# Single-Node Real-Time Data Runtime

Language-agnostic project skeleton for a single-node real-time data pipeline.
No application code or runtime dependencies are included yet.

## Directory layout

- `config/`: runtime and pipeline configuration.
- `data/input/`: incoming source data.
- `data/work/`: transient processing data.
- `data/output/`: processed results.
- `data/archive/`: retained input or output data.
- `logs/`: application and operational logs.
- `run/`: runtime state such as PID or lock files.
- `scripts/`: start, stop, and maintenance entry points.
- `src/ingest/`: data collection and input adapters.
- `src/process/`: parsing, transformation, and processing logic.
- `src/output/`: output adapters and delivery logic.
- `tests/`: unit and integration tests.

## Runtime flow

`data/input/` → `src/ingest/` → `src/process/` → `src/output/` → `data/output/`

Add the chosen language's entry point and launch commands under `scripts/` once
the runtime and data source are specified. Keep transient runtime state in
`run/` and operational logs in `logs/`.
