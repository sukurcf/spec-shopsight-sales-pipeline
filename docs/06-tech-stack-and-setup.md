# Tech stack and setup

Purpose: This document defines the approved ShopSight tools, local setup steps, hardware profiles, accounts, and learning order.

## Reference stack

Students MUST use these reference minor lines unless an allowed alternative says that an ADR is required.

| Category | Tool | Reference version | Purpose | Why this tool |
|---|---|---:|---|---|
| Language | Python | 3.12.x | Simulator, ingestion, FX client, report/local CLIs, tests, and Airflow code. | Cohort baseline with support across Airflow 3 and dbt tooling. |
| Package manager | uv | 0.12.x | Python environments and lockfiles. | Fast installs and reproducible `uv.lock`. |
| Data frames | pandas or Polars | pandas 3.0.x or Polars 1.44.x | Read, validate, and split CSV data. | Both handle Olist-sized CSV files locally. |
| Validation | pandera | 0.33.x | Source schema and row validation. | Clear dataframe validation before PostgreSQL load. |
| Database | PostgreSQL | 18.x | Raw, quarantine, staging, marts, Airflow metadata. | Free relational warehouse for SQL and dbt practice. |
| Database driver | psycopg | 3.3.x | Bulk database loading from Python. | Modern PostgreSQL driver with COPY support. |
| Transformations | dbt Core and dbt-postgres | dbt-core 1.12.x and dbt-postgres 1.11.x | Staging, intermediate, mart, tests, docs. | Standard analytics engineering tool for PostgreSQL. |
| Orchestration | Apache Airflow | 3.3.x | Daily DAG, backfill, retries, sensors, alerts. | Industry-standard batch orchestrator. |
| Containers | Docker Engine | 29.x | Local service runtime. | One reproducible environment for PostgreSQL and Airflow. |
| Compose | Docker Compose | v5.x | Multi-service local startup. | Official Airflow Compose uses this model. |
| Mail sink | Mailpit | 1.31.x | Failure-alert testing. | Maintained replacement for MailHog. |
| Tests | pytest | 9.1.x | Python unit and integration tests. | Common entry-level Python test runner. |
| Coverage | pytest-cov and coverage | 7.1.x and 7.16.x | Enforce 85% line and 75% branch coverage. | CI can fail below exact thresholds. |
| Lint and format | Ruff | 0.16.x | Python linting and formatting. | Fast single tool for style checks. |
| Type check | mypy | 2.4.x | Typed Python boundaries. | Catches bad optional and untyped functions. |
| SQL lint | SQLFluff | 4.3.x | dbt SQL linting. | Teaches readable SQL style. |
| Git hooks | pre-commit | 4.6.x | Local quality checks before commit. | Reduces broken PRs. |
| Dependency scan | pip-audit | 2.10.x | Python vulnerability scan. | Free CI security gate. |
| Container scan | Trivy | 0.75.x | Container and dependency scanning. | Detects known image vulnerabilities. |
| Secret scan | Gitleaks | 8.30.x | Detect committed secrets. | Mandatory before public portfolio work. |

## Allowed alternatives that require an ADR

| Decision | Allowed alternative | ADR question to answer |
|---|---|---|
| Data frame engine | pandas or Polars | Which engine gives simpler validation and acceptable runtime for 100,000 orders? |
| dbt engine | dbt v2 | What changed in dbt v2, what risk exists, and why is it worth using before the cohort default? |
| Quality tool | Great Expectations or Soda as Could scope | What extra checks does it add beyond pandera and dbt tests? |
| Lake layer | DuckDB plus Parquet as Could scope | What query or performance value does the local lake add? |

## Not allowed

| Tool or practice | Reason |
|---|---|
| Docker images tagged `latest` | Rebuilds can silently change behaviour. |
| Bitnami images or charts | The free catalog changed in 2025. Do not depend on it. |
| MailHog | It is unmaintained. Use Mailpit. |
| Mandatory MinIO | Local object storage is not needed for Must scope. |
| Cloud-deployment or paid-account exercises | The project runs locally; these are not optional or bonus scope. |
| Floating point money | BR-17 requires decimal money. |
| Live Frankfurter calls in CI | FR-DQ-01 requires recorded fixtures. |
| Student-built frontend or BI interfaces | No frontend work is allowed at any priority. Built-in Airflow/Mailpit consoles may only be inspected for operations. |

## Local setup checklist

### Windows 11 with WSL2 Ubuntu 24.04

1. Enable WSL2 and install Ubuntu 24.04.
2. Install Docker Desktop and select the WSL2 backend.
3. Create `.wslconfig` in your Windows user folder with the values below.
4. Install Git, uv 0.12.x, and VS Code.
5. Install VS Code extensions for Python, Ruff, Docker, dbt, and Markdown.
6. Clone your student repository named `shopsight`.
7. Add the trainer account `@sukurcf` as a collaborator.
8. Complete initial dependency/image downloads and prepare recorded historical FX fixtures in the student repository.
9. Follow the single start entry point below; it generates fictional Olist-equivalent seed data locally.
10. Run the offline demonstration and stop through the documented entry point. Kaggle input is optional.

### `.wslconfig` values

Use the same recommendation for every local run:

| Host RAM | WSL memory | WSL swap |
|---|---|---|
| 8 GB | 4 GB | 4 GB |
| 16 GB | 8 GB | 4 GB |

Allocate 4 processors on a four-core host; do not allocate more processors than the host has.

### macOS

1. Install Docker Desktop for Mac or another Docker Engine with Compose v5.
2. Install Homebrew if you use it for Git and uv.
3. Install Git, uv 0.12.x, and VS Code.
4. Keep at least 15 GB of free disk for Docker volumes.
5. Use the standard profile on 16 GB machines.
6. Use the lite profile on 8 GB machines.

### Linux

1. Install Docker Engine 29.x and Docker Compose v5.
2. Add your user to the Docker group only if your distribution recommends it.
3. Install Git, uv 0.12.x, and VS Code or another editor.
4. Confirm that the fixed loopback ports below are free.
5. Use the lite profile if the machine has 8 GB RAM.

## Local operation contract students MUST implement

These are interfaces for the future student implementation, not commands supplied by this specification repository. They apply to Windows 11 with WSL2 Ubuntu, macOS, and Linux.

Initial locked Python installs (`uv sync --frozen`), image pulls/builds, and tool downloads need internet. GitHub submission and hosted CI need internet. After those downloads, startup, the seeded demonstration, runtime tests, and report exports MUST work without external network access. No paid account, public hostname, API key, or cloud resource is required. GitHub is not a local runtime dependency.

### Single start and stop entry points

| Command, run from the student repository | Contract |
|---|---|
| Start: `uv run --no-sync shopsight local start --profile lite` | The one default start entry point. Preflight Docker, disk, configuration, and ports; initialize or reuse persistent services; wait for health. Defaults: `SHOP_SOURCE_MODE=synthetic`, `SHOP_FX_MODE=fixture`. Exit `0` only after all services are healthy, and print the exact healthy JSON below. |
| Stop: `uv run --no-sync shopsight local stop` | The one stop entry point. Gracefully stop project services; never delete volumes, seeds, landing files, or reports. Return `{"status":"stopped","data_preserved":true}`. |
| Health: `uv run --no-sync shopsight local health --format json` | Query local database, Airflow health/heartbeats, and Mailpit availability without external HTTP. Exit `0` only when all are healthy. |
| Demonstrate: `uv run --no-sync shopsight demo --fixture fixture_2018_01_02_small` | Trigger the historical fixture through the same raw/FX/dbt/quality path used by the DAG; emit the exact summary below. |
| Confirmed reset: `uv run --no-sync shopsight local reset --confirm-delete-data` | Stop services and delete only this project's local volumes/generated data. Refuse without the exact confirmation flag. Never delete source code or committed fixtures. |

Start MUST be idempotent. It creates the warehouse and separate Airflow metadata database, versioned raw/audit/committed-state schema migrations, restricted loader/transform/report users, and Airflow metadata migrations. It registers the daily DAG paused, avoiding automatic historical catchup. It locally generates fictional Olist-equivalent bootstrap/daily seed files using seed `20261002` and initializes fixture-mode configuration. Existing data is not overwritten on restart. Standard profile uses the same start command with `--profile standard`, not a second start mechanism.

### Fixed loopback ports

| Service | Host binding | Internal service port | Health evidence |
|---|---|---:|---|
| PostgreSQL warehouse and metadata | `127.0.0.1:15434` | 5432 | `SELECT 1`, required schemas, migration version, and restricted users exist. |
| Airflow api-server | `127.0.0.1:18080` | 8080 | `/api/v2/monitor/health` plus scheduler and DAG-processor heartbeat checks. |
| Mailpit SMTP | `127.0.0.1:11027` | 1025 | Local SMTP connection succeeds; captured alerts can be verified through Mailpit's API. |
| Mailpit API / built-in operations console | `127.0.0.1:18027` | 8025 | Local API is reachable; console inspection is optional. |

These host ports differ from CampusHire and VayuStream's documented ports. Separate port assignments do not guarantee enough RAM for concurrent projects. Do not publish scheduler, DAG processor, or container-internal database ports separately. Bind every published port to `127.0.0.1`, not `0.0.0.0`. An occupied port produces `LOCAL-PORT-IN-USE`, exit `3`, without changing the fixed mapping. Airflow and Mailpit consoles are operations-only tools, never mandatory business evidence.

Healthy default-local-mode output is exactly:

```json
{"status":"healthy","services":{"postgres":"healthy","airflow":"healthy","mailpit":"healthy"},"source_mode":"synthetic","fx_mode":"fixture"}
```

Successful start and the health command MUST each print this single JSON object to stdout. Send progress messages to stderr. On startup failure, print the documented error to stderr and exit nonzero. Do not print a successful health object.

### Deterministic offline demonstration and fixtures

The named fixture has 9 source files and `63` source rows, `60` accepted rows, and `3` quarantined rows. It builds `7` delivered mart orders and `9` items. Recorded historical FX fixture rate for `2018-01-02` is `19.50` INR per BRL. Demonstration output is exactly:

```json
{"fixture":"fixture_2018_01_02_small","source_rows":63,"accepted_rows":60,"quarantined_rows":3,"order_count":7,"item_count":9,"gmv_brl":"900.00","gmv_inr":"17550.00","freight_brl":"120.00"}
```

The demo summarizes the current committed fixture population, not cumulative audit attempts. An unchanged rerun retains these totals while recording new skipped attempts; it does not sum earlier attempts into source/accepted/quarantine counts.

Run `uv run --no-sync shopsight report export --view mart_monthly_gmv --month 2018-01 --format json`, then repeat with `--format csv`. Exact envelopes, headers, row values, and all six view contracts are in [doc 08](08-pipeline-specification.md). `--output reports/monthly.csv` writes an atomic file instead of stdout. No UI is started.

Commit the small source fixtures, correction versions A/B/A, failing responses, and recorded historical FX files under `data/fixtures/`; include recording provenance (source URL, provider, requested dates, preceding market-day seed, and checksum). Recordings MUST cover the selected synthetic historical generator's 2016-2018 date range and its preceding market-day seed, not just the one-day demo. Default local mode uses them by configuration, never after live failure. Missing recordings fail with `FX-FIXTURE-MISSING`; health mode fields reflect configuration in opt-in modes.

Production-like input is separate and opt-in: the same start entry point accepts `--source-mode olist --fx-mode live` after the user downloads real Olist input. Live Frankfurter obeys the ECB/date/range rules, 2 retries, 5-minute delay, and terminal failure alert. It MUST NOT switch to synthetic data or fixture FX on error. Tests of live retry behavior use recorded failing HTTP responses, not internet.

### Persistence and failure acceptance

Use project-scoped named volumes `shopsight_pg_data` and `shopsight_airflow_logs`; bind mounts for generated data, landing files, and report outputs survive stop/start. Ignore generated runtime data in Git. Reset explicitly deletes only these project resources; it does not affect VayuStream or other projects. A port collision, invalid profile/view/filter, missing fixture, unavailable Docker, or unavailable PostgreSQL has a stable stderr error code and nonzero exit, with no partial financial report.

Trainer pre-check and grading MUST use `TC-LOCAL-001` through `TC-LOCAL-004` in [doc 09](09-testing-strategy-and-test-cases.md), including offline execution and persisted checksum/count comparisons.

## Hardware profiles

| Profile | Laptop target | Active services | Exact limits |
|---|---|---|---|
| Lite | 4 CPU cores, 8 GB RAM, 15 GB free disk | PostgreSQL 18, Airflow 3 scheduler, api-server, DAG processor, Mailpit; short-lived CLI | WSL memory 4 GB and swap 4 GB. Airflow parallelism 1, max active DAG runs 1, dbt threads 1, PostgreSQL shared buffers at most 128 MB. |
| Standard | 6 CPU cores, 16 GB RAM, 25 GB free disk recommended | Same services; optional triggerer for deferrable operators | WSL memory 8 GB and swap 4 GB. Airflow parallelism 4, max active DAG runs 2. |

The official Airflow Docker Compose file MUST be adapted for Airflow 3.3.x and LocalExecutor. It MUST use scheduler, api-server, DAG processor, and metadata PostgreSQL. It MUST NOT add Celery workers, Redis, or webserver guidance. The lite profile uses ordinary sensors and no triggerer. API-server screens are built-in operations tools only; students do not implement or style them.

### Proposed aggregate RAM budgets

| Service or task | Proposed lite limit | Proposed standard limit |
|---|---:|---:|
| PostgreSQL warehouse and Airflow metadata, one container | 768 MB | 1536 MB |
| Airflow scheduler and LocalExecutor child tasks combined | 1280 MB | 2560 MB |
| Airflow api-server | 384 MB | 768 MB |
| Airflow DAG processor | 384 MB | 768 MB |
| Airflow triggerer, only for deferrable operators | Not started | 256 MB |
| Mailpit | 64 MB | 128 MB |
| Short-lived report/local CLI | 256 MB | 512 MB |
| Total at stated concurrency | 3136 MB | 6528 MB including optional triggerer |

These are proposed design limits, not measured results. The scheduler budget includes its loader/dbt child task; do not count that task a second time. The lite total leaves about 960 MB within a 4 GB WSL/VM allocation for Linux/container overhead; the host retains the other 4 GB. Trainer pre-check MUST record actual peak RAM, swap, and elapsed time on an 8 GB host before approving the implemented profile. If it does not fit, reduce concurrency or fixture size without removing Must behavior.

## Trainer pre-check

Before week 1, the trainer MUST validate these items on an 8 GB laptop.

| Check | Expected result |
|---|---|
| Docker starts PostgreSQL, Airflow, and Mailpit | Services become healthy without swap storms. |
| Airflow imports the daily DAG | No import errors and no cycles. |
| A one-day sample load runs | Accepted plus quarantined rows equal source rows. |
| dbt build runs on sample data | Required marts and tests complete. |
| Offline report export | Monthly JSON/CSV exactly matches doc 08; all six views are readable without a browser. |
| Stop/start and failure suite | TC-LOCAL-001 through TC-LOCAL-004 pass without data loss or external HTTP. |
| Loopback bindings and RAM | Ports match this document; measured student evidence is compared to the proposed budgets, not assumed. |
| CI sample is small | Each source file has at most 1,000 data rows. |

## Free accounts needed

| Account | Required? | Purpose |
|---|---|---|
| GitHub | Must | Student repository, issues, PRs, CI. |
| Kaggle | Optional, real-source mode only | Download real Olist input; never needed for the synthetic local profile. |
| Docker Hub | Should | Avoid anonymous pull limits. |
| Frankfurter | No account | FX API is free and keyless. |

## Suggested learning order

| Order | Topic | Hours | Outcome |
|---:|---|---:|---|
| 1 | GitHub Flow and Conventional Commits | 3 | Open small PRs and explain commits. |
| 2 | Docker Compose basics | 4 | Start PostgreSQL and Mailpit locally. |
| 3 | PostgreSQL joins, GROUP BY, CTEs, windows, indexes | 8 | Query Olist-style facts and dimensions. |
| 4 | Python CSV ingestion and pandera | 7 | Validate a daily folder and quarantine bad rows. |
| 5 | dbt staging, marts, tests, docs | 8 | Build a tested star schema. |
| 6 | Airflow DAGs, sensors, retries, backfill, XCom | 6 | Explain the daily DAG and failures. |
| 7 | Python CLI serialization and offline reliability | 3 | Export six views as exact JSON/CSV and verify restart/error contracts. |
| 8 | CI, coverage, and security scans | 4 | Read and fix a failing PR check. |
| 9 | Interview revision | 7 | Explain ETL versus ELT, idempotency, and model grains. |

[Back to README](../README.md)
