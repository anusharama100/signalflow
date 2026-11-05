# Local verification

Checked November 5, 2026 with Python 3.12 on macOS and a separate PostgreSQL 16 test container.

- Installed `requirements.lock` into a fresh virtual environment.
- All 37 tests passed with SQLite, then all 37 passed with a separate PostgreSQL test database.
- Ingested seven days from the live NESO API into a new local SQLite database: 672 accepted events, with no quarantined events.
- The populated API, interval CSV export, and workload endpoint returned HTTP 200.
- Browser walkthrough: inspected the dashboard and changed IT power from 100 to 200 kW. Facility energy changed from 240 to 480 kWh and calculated emissions doubled. No console errors were observed.

Live-source values change. The Kafka, Airflow, and dbt extensions were not rerun in this check; their earlier integration evidence remains in `docs/`.
