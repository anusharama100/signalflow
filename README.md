# SignalFlow

A pipeline and dashboard for Great Britain's grid carbon intensity and generation mix. SignalFlow keeps forecasts and observed results separate, preserves source revisions, and lets you compare the emissions of running the same electrical workload at different historical times.

The interesting part is the changing data: actual readings arrive late, upstream values are revised, and a consumer can receive the same event twice. The project records those changes without turning missing observations into zero or counting an exact replay as new data.

![SignalFlow dashboard](docs/screenshots/dashboard.png)

[Case study](docs/case-study.md) · [Architecture](docs/architecture.md) · [Verification](docs/verification.md) · [CI runs](https://github.com/anusharama100/signalflow/actions)

## What it includes

- Collection from the public NESO Carbon Intensity API, with timeouts and bounded retries.
- Raw source events, content hashes, validation decisions, and interval-level projections.
- Updates for delayed actuals and revised generation mixes; exact replay is a no-op.
- A dashboard for intensity, forecast error, renewable share, missing values, freshness, and CSV exports.
- A historical workload comparison using IT power, duration, and power usage effectiveness (PUE).
- A simple SQLite path, plus optional PostgreSQL, Kafka, dbt, and Airflow services.

## Quick start

Use Python 3.12 and `make`. This path uses SQLite; no Docker or API key is needed.

```sh
git clone https://github.com/anusharama100/signalflow.git
cd signalflow
make setup
make ingest
make serve
```

Open [localhost:8000](http://127.0.0.1:8000). `make ingest` collects the preceding seven days and creates a local database under the ignored `data/` directory. Start `make worker` in another terminal to revisit the preceding two days every five minutes.

Without `make`:

```sh
python3 -m venv .venv
.venv/bin/pip install -r requirements.lock
.venv/bin/python -m signalflow.cli ingest --days 7
.venv/bin/python -m signalflow.cli serve
```

The dashboard refreshes stored results every minute. It does not trigger ingestion. Source observations cover half-hour intervals.

## Optional container stack

For PostgreSQL with direct polling:

```sh
docker compose --profile polling up --build -d
docker compose run --rm api python -m signalflow.cli ingest --days 7
```

For Kafka ingestion, use `docker compose --profile streaming up --build -d` instead. Choose one routine ingestion path to avoid redundant upstream requests. Add the `orchestration` profile for Airflow and dbt. The [runbook](docs/runbook.md) covers service commands, generated local Airflow credentials, configuration, and recovery.

## Design decisions

**Python, FastAPI, SQLAlchemy, HTTPX, SQLite/PostgreSQL, Kafka, dbt, and Airflow.** The dashboard uses HTML, CSS, JavaScript, and SVG without a frontend build step.

Raw events and their analytical projections commit in one database transaction. A content hash identifies an exact replay. A changed payload remains a separate event and can update its interval; forecast-only records cannot erase stored actuals.

The Kafka consumer commits offsets after database processing. A crash between those steps can replay the event, which the database deduplicates. Database writes and Kafka offsets do not share a transaction; dead-letter delivery can also repeat.

| Directory | Contents |
| --- | --- |
| `signalflow/` | Collection, materialization, analytics, CLI, API, and dashboard |
| `analytics/` | dbt staging, daily mart, and tests |
| `orchestration/` | Airflow reconciliation DAG and isolated environments |
| `tests/` | Deterministic tests and source fixtures |
| `docs/` | Architecture, SQL, screenshots, and execution evidence |

API entry points include `/api/overview`, `/api/scenario`, `/api/export.csv`, `/health`, `/metrics`, and interactive documentation at `/docs`.

## Verification

Run `make test` for the deterministic suite and `make check` against an ingested dataset. The recorded [verification](docs/verification.md) includes:

- 37 tests against SQLite and a separate PostgreSQL test database.
- A seven-day backfill with 672 distinct source events and 336 intensity intervals; exact replay added no events.
- A real Kafka replay reaching zero consumer lag on all three partitions.
- Two dbt models, 12 passing data tests, and a successful Airflow reconciliation.

These counts describe captured development runs, not a fixed live feed. [Container integration evidence](docs/integration-evidence.json) is separate from the unit tests' Kafka test doubles.

## Reading the results

Intensity is grams of CO2 per kWh. Renewable share includes wind, solar, and hydro. Forecast error excludes missing actuals; coverage describes stored intervals, not proof that every expected interval arrived.

The workload comparison uses contiguous, fully observed half-hour windows:

```text
Energy (kWh) = IT power (kW) × duration (hours) × PUE
Emissions (kg CO2) = energy × mean intensity / 1000
```

It is a hindsight comparison, not a forecast or a promise of savings. It excludes electricity prices, embodied emissions, and site-specific contracts. Anomaly flags are readings to investigate, not confirmed grid incidents.

## Current scope

This is a local development deployment. Sustained throughput, high availability, cloud hosting, and recovery from infrastructure loss have not been demonstrated. Hosting requires access controls, managed secrets, schema migrations, monitoring, and tested backups. The single Kafka broker is not highly available.

Data comes from the [NESO Carbon Intensity API](https://api.carbonintensity.org.uk/) under CC BY 4.0. Preserve its attribution when sharing extracts. See the [official endpoint definitions](https://carbon-intensity.github.io/api-definitions/) and [architecture notes](docs/architecture.md).
