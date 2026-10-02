# Project overview

Purpose: This document explains why ShopSight exists, what the student must build, and how success will be measured.

## Problem statement

An online marketplace uses the Olist Brazilian e-commerce data as its training business data. Analysts currently join daily CSV exports in spreadsheets. That manual process creates duplicate rows, missed bad rows, and inconsistent revenue totals.

The Indian investor board wants daily sales metrics in INR. Source orders are priced in BRL. ShopSight MUST build a local Python batch ELT pipeline that validates daily CSV drops, loads PostgreSQL, transforms data with dbt, and publishes trusted sales marts and JSON/CSV reports.

## Vision

ShopSight is a portfolio-grade data engineering project for one fresher. It proves the student can ingest files, protect data quality, model a warehouse, orchestrate a batch pipeline, and explain design trade-offs in a viva.

## Measurable goals

| Goal | Target | Evidence |
|---|---:|---|
| Historical load size | About 100,000 synthetic Olist-equivalent orders from 2016 to 2018; real Olist is opt-in | Local backfill run log |
| Daily run time | SHOULD finish in less than 5 minutes after setup | Timed Airflow run |
| Full backfill time | SHOULD finish in less than 30 minutes | Timed backfill run |
| Python coverage | At least 85% line and 75% branch | CI coverage report |
| dbt key tests | 100% of mart primary keys have unique and not-null tests | dbt test report |
| dbt relationship tests | 100% of mart foreign keys have relationship tests | dbt test report |
| Test catalog size | At least 50 test cases | Testing document |
| Report scope | Exact JSON/CSV exports from all 6 required analytics views | CLI snapshot tests and final demo |

## Effort budget

The full project budget is about 180 hours over 6 weeks. The Must scope is limited to about 120 hours. This includes about 30 hours of guided learning and setup in week 1. The remaining 60 hours are for Should items, hardening, interview preparation, and contingency.

| Must area | Budget |
|---|---:|
| Guided learning and setup | 30 hours |
| Simulator and sample data | 10 hours |
| Raw ingestion, validation, audit, and quarantine | 18 hours |
| Exchange-rate ingestion and fixtures | 8 hours |
| dbt transformations and marts | 20 hours |
| Data quality and reconciliation tests | 10 hours |
| Airflow orchestration and alerts | 10 hours |
| SQL analytics, Python exports, and local reliability | 8 hours |
| CI and student documentation | 6 hours |
| **Total Must scope** | **120 hours** |

## Scope by priority

| Priority | Scope |
|---|---|
| Must | Explicit local synthetic daily-drop mode, raw load with quarantine and audit, recorded historical BRL to INR fixtures plus opt-in live FX, dbt star schema, money reconciliation, Airflow daily DAG, 6 analytics views, Python JSON/CSV exports, offline startup and persistence tests, CI, runbook, Olist attribution. |
| Should | Configurable quality-problem rates, deterministic late status updates, customer SCD Type 2, 2 extra analytics views, per-run Markdown/JSON/CSV quality report, performance targets. |
| Could | Local Parquet lake with DuckDB, Great Expectations or Soda, Astronomer Cosmos, locally captured OpenLineage evidence. |

## Non-goals and out of scope

- Streaming ingestion is out of scope.
- Machine learning is out of scope.
- Cloud-deployment exercises and cloud warehouses are out of scope.
- Student-built frontends, charts, and BI interfaces are out of scope at every priority. The built-in Airflow console is optional operations inspection only.
- Real payment processing is out of scope.
- Real personal data is out of scope.
- Production high availability is out of scope.
- Exactly-once e-mail or webhook delivery is out of scope.

## Assumptions

- The trainer confirms that Olist use is non-commercial training.
- The Olist dataset licence is CC BY-NC-SA 4.0.
- The student locally generates synthetic data with all 9 Olist file names, real column names, and key relationships.
- Real Kaggle/Olist input is an explicitly selected source mode, not required for startup or grading.
- The project uses Python 3.12.x and local free tools.
- Local runtime, tests, and CI default to recorded historical FX fixtures. Live mode is explicit and still fails/alerts after exhausted retries; it never switches to fixtures.
- `customer_unique_id` is the customer business key.
- `customer_id` is only a source join key for orders.

## Constraints

| Constraint | Requirement |
|---|---|
| Hardware | Must scope MUST run on 8 GB RAM with the lite profile. |
| Lite profile | Single active DAG run, one LocalExecutor task at a time, and no UI services beyond optional built-in operations consoles. Proposed budgets are defined in doc 06. |
| Money | Money MUST use decimals, not floats. |
| Rounding | Money MUST round to 2 decimals only at mart output level. |
| Revenue | GMV MUST equal sum of item prices. Freight is separate. |
| FX | No silent FX fallback is allowed after retries fail. |
| Idempotency | Each logical date can be rerun without duplicated accepted rows. |
| Licence | Olist attribution and CC BY-NC-SA 4.0 terms MUST be documented. |

## Success criteria

1. The student can start the project from a clean clone using doc 06, after initial downloads, without external network access.
2. CI is green on `main`.
3. The pipeline loads accepted and quarantined source rows.
4. Each load proves `source rows = accepted rows + quarantined rows`.
5. Rerunning one logical date with unchanged input checksums keeps row counts and totals unchanged.
6. Marts publish `fct_order_items`, `fct_orders`, and required dimensions.
7. The Python CLI exports the six views with exact columns, decimal strings, stable ordering, and empty-result behavior.
8. Stop/start preserves data; invalid input and unavailable local dependencies produce clear errors.
9. The student can explain one failure and its fix in the viva.

## Skills learned and job relevance

| Skill | Job-description keywords |
|---|---|
| Batch ingestion | Python ETL, CSV ingestion, bulk load, file checksum |
| Data validation | pandera, schema validation, quarantine, audit table |
| Data warehouse | PostgreSQL, star schema, fact table, dimension table |
| Analytics engineering | dbt, staging, intermediate, marts, lineage, tests |
| Orchestration | Airflow DAG, sensor, retry, timeout, backfill |
| Data reliability | Idempotency, reconciliation, source freshness, fixtures |
| SQL analytics | Joins, GROUP BY, CTEs, window functions, indexes |
| DevOps | Docker Compose, GitHub Actions, coverage, security scans |

[Back to README](../README.md)
