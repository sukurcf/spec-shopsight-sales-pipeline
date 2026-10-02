# Tech stack and setup

Purpose: This document defines the approved ShopSight tools, local setup steps, hardware profiles, accounts, and learning order.

## Reference stack

Students MUST use these reference minor lines unless an allowed alternative says that an ADR is required.

| Category | Tool | Reference version | Purpose | Why this tool |
|---|---|---:|---|---|
| Language | Python | 3.12.x | Simulator, ingestion, FX client, tests, and Airflow code. | Cohort baseline with support across Airflow 3 and dbt tooling. |
| Package manager | uv | 0.12.x | Python environments and lockfiles. | Fast installs and reproducible `uv.lock`. |
| Data frames | pandas or Polars | pandas 3.0.x or Polars 1.44.x | Read, validate, and split CSV data. | Both handle Olist-sized CSV files locally. |
| Validation | pandera | 0.33.x | Source schema and row validation. | Clear dataframe validation before PostgreSQL load. |
| Database | PostgreSQL | 18.x | Raw, quarantine, staging, marts, Airflow metadata. | Free relational warehouse for SQL and dbt practice. |
| Database driver | psycopg | 3.3.x | Bulk database loading from Python. | Modern PostgreSQL driver with COPY support. |
| Transformations | dbt Core and dbt-postgres | dbt-core 1.12.x and dbt-postgres 1.11.x | Staging, intermediate, mart, tests, docs. | Standard analytics engineering tool for PostgreSQL. |
| Orchestration | Apache Airflow | 3.3.x | Daily DAG, backfill, retries, sensors, alerts. | Industry-standard batch orchestrator. |
| Dashboard | Streamlit or Metabase OSS | Streamlit 1.64.x or Metabase 0.63.x | 4-chart sales dashboard. | Streamlit is lighter for 8 GB laptops. Metabase is BI-like. |
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
| Dashboard | Metabase instead of Streamlit, or Streamlit instead of Metabase | Which tool fits the 8 GB lite profile and the 4 Must charts best? |
| Data frame engine | pandas or Polars | Which engine gives simpler validation and acceptable runtime for 100,000 orders? |
| dbt engine | dbt v2 | What changed in dbt v2, what risk exists, and why is it worth using before the cohort default? |
| Quality tool | Great Expectations or Soda as Could scope | What extra checks does it add beyond pandera and dbt tests? |
| Lake layer | DuckDB plus Parquet as Could scope | What query or performance value does the local lake add? |
| Cloud warehouse | Free-tier warehouse as Could scope | What is the cost limit, and how will you destroy it after the demo? |

## Not allowed

| Tool or practice | Reason |
|---|---|
| Docker images tagged `latest` | Rebuilds can silently change behaviour. |
| Bitnami images or charts | The free catalog changed in 2025. Do not depend on it. |
| MailHog | It is unmaintained. Use Mailpit. |
| Mandatory MinIO | Local object storage is not needed for Must scope. |
| Mandatory cloud services | Must scope must run locally and free. |
| Floating point money | BR-17 requires decimal money. |
| Live Frankfurter calls in CI | FR-DQ-01 requires recorded fixtures. |
| Running dashboard with Airflow on lite profile | BR-25 forbids this on 8 GB laptops. |

## Local setup checklist

### Windows 11 with WSL2 Ubuntu 24.04

1. Enable WSL2 and install Ubuntu 24.04.
2. Install Docker Desktop and select the WSL2 backend.
3. Create `.wslconfig` in your Windows user folder with the values below.
4. Install Git, uv 0.12.x, and VS Code.
5. Install VS Code extensions for Python, Ruff, Docker, dbt, and Markdown.
6. Clone your student repository named `shopsight`.
7. Add the trainer account `@sukurcf` as a collaborator.
8. Download the Olist dataset locally. Do not commit the full data.
9. Start only PostgreSQL, Airflow, and Mailpit for the lite profile.
10. Start the dashboard only after Airflow is stopped on an 8 GB laptop.

### `.wslconfig` values

For an 8 GB laptop, use this cohort decision: memory `4GB`, swap `4GB`, processors `4`.
For a 16 GB laptop, use memory `8GB`, swap `4GB`, processors `6`.
Change these values only after writing the reason in the student README.

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
4. Confirm that ports for PostgreSQL, Airflow, Mailpit, and dashboard are free.
5. Use the lite profile if the machine has 8 GB RAM.

## Hardware profiles

| Profile | Laptop target | Active services | Exact limits |
|---|---|---|---|
| Lite | 4 CPU cores, 8 GB RAM | PostgreSQL 18, Airflow 3 scheduler, api-server, DAG processor, Mailpit | WSL memory 4 GB and swap 4 GB. Airflow parallelism 2. Max active DAG runs 1. PostgreSQL shared buffers no more than 512 MB. Dashboard is on demand only. |
| Standard | 6 CPU cores, 16 GB RAM | PostgreSQL, Airflow scheduler, api-server, DAG processor, Mailpit, dashboard | WSL memory 8 GB and swap 4 GB. Airflow parallelism 4. Max active DAG runs 2. Dashboard may stay active during analysis. |

The official Airflow Docker Compose file MUST be adapted for Airflow 3.3.x and LocalExecutor. It MUST use scheduler, api-server, DAG processor, and metadata PostgreSQL. It MUST NOT add Celery workers, Redis, or webserver guidance. A triggerer is needed only for deferrable operators. The lite profile MUST lower parallelism. The dashboard MUST NOT run together with Airflow on the lite profile.

### Per-service RAM ceilings

| Service or task | Lite ceiling | Standard ceiling |
|---|---:|---:|
| PostgreSQL warehouse and Airflow metadata | 1024 MB | 2048 MB |
| Airflow scheduler | 512 MB | 1024 MB |
| Airflow api-server | 512 MB | 1024 MB |
| Airflow DAG processor | 512 MB | 1024 MB |
| Airflow triggerer, only for deferrable operators | 256 MB | 512 MB |
| dbt run inside an Airflow task | 1024 MB | 2048 MB |
| Mailpit | 128 MB | 256 MB |
| Dashboard, on demand in lite profile | 512 MB | 1024 MB |

The trainer pre-check MUST validate these ceilings on an 8 GB laptop.

## Trainer pre-check

Before week 1, the trainer MUST validate these items on an 8 GB laptop.

| Check | Expected result |
|---|---|
| Docker starts PostgreSQL, Airflow, and Mailpit | Services become healthy without swap storms. |
| Airflow imports the daily DAG | No import errors and no cycles. |
| A one-day sample load runs | Accepted plus quarantined rows equal source rows. |
| dbt build runs on sample data | Required marts and tests complete. |
| Dashboard starts after Airflow stops | Four chart placeholders render. |
| CI sample is small | Each source file has at most 1,000 data rows. |

## Free accounts needed

| Account | Required? | Purpose |
|---|---|---|
| GitHub | Must | Student repository, issues, PRs, CI. |
| Kaggle | Must unless fallback is used | Download the Olist dataset. |
| Docker Hub | Should | Avoid anonymous pull limits. |
| Frankfurter | No account | FX API is free and keyless. |
| Cloud provider | Could only | Optional warehouse demo with cost warning. |

## Suggested learning order

| Order | Topic | Hours | Outcome |
|---:|---|---:|---|
| 1 | GitHub Flow and Conventional Commits | 3 | Open small PRs and explain commits. |
| 2 | Docker Compose basics | 4 | Start PostgreSQL and Mailpit locally. |
| 3 | PostgreSQL joins, GROUP BY, CTEs, windows, indexes | 8 | Query Olist-style facts and dimensions. |
| 4 | Python CSV ingestion and pandera | 7 | Validate a daily folder and quarantine bad rows. |
| 5 | dbt staging, marts, tests, docs | 8 | Build a tested star schema. |
| 6 | Airflow DAGs, sensors, retries, backfill, XCom | 6 | Explain the daily DAG and failures. |
| 7 | Dashboard basics | 3 | Build the four Must charts. |
| 8 | CI, coverage, and security scans | 4 | Read and fix a failing PR check. |
| 9 | Interview revision | 7 | Explain ETL versus ELT, idempotency, and model grains. |

[Back to README](../README.md)
