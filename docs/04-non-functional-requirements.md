# Non-functional requirements

Purpose: This document defines measurable quality targets for ShopSight.

## NFR catalog

| ID | Title | Priority | Measurable target | Verification method |
|---|---|---|---|---|
| NFR-PERF-01 | Full backfill time | Should | About 100,000 generated Olist-equivalent orders SHOULD backfill offline in less than 30 minutes on the standard profile; real Olist is opt-in. | Timed local demo with clean volumes and recorded FX. |
| NFR-PERF-02 | Daily run time | Should | One logical date SHOULD finish in less than 5 minutes after setup. | Timed Airflow run on a representative date. |
| NFR-PERF-03 | CI duration | Must | Pull-request CI MUST finish in 15 minutes or less with sample data. | GitHub Actions run duration. |
| NFR-SCALE-01 | Local data volume | Must | Must scope MUST handle about 100,000 locally generated Olist-equivalent orders and CI samples of at most 1,000 rows per source table; real Olist is opt-in. | Synthetic backfill log and CI data check. |
| NFR-REL-01 | Logical-date idempotency | Must | Running the same logical date twice with unchanged input checksums MUST keep accepted counts and mart totals identical. | Integration test. |
| NFR-REL-02 | Retry policy | Must | Airflow tasks that touch files, database, FX, dbt, or alerts MUST use 2 retries and 5-minute retry delay. | DAG integrity test. |
| NFR-REL-03 | FX failure safety | Must | If Frankfurter fails after retries, the run MUST fail and alert. It MUST NOT write a made-up rate. | Fault-injection test. |
| NFR-SEC-01 | Secret handling | Must | Secrets MUST come from environment variables and MUST NOT appear in Git history. | Gitleaks scan and PR review. |
| NFR-SEC-02 | Database least privilege | Must | Separate local users MUST serve raw load, transformations, report reads, and Airflow metadata. The report user can select the six analytics views only, not read raw schemas or write marts. | Configuration review and TC-IT-011. |
| NFR-SEC-03 | Dependency scanning | Must | CI MUST run Python dependency and container vulnerability scans. | CI gate. |
| NFR-PRIV-01 | Personal-data awareness | Must | The student MUST document that customer IDs and location fields can become personal data under DPDP Act 2023 principles. | Documentation review. |
| NFR-MAINT-01 | Typed Python boundary | Must | Application packages MUST pass mypy with `disallow_untyped_defs` and `no_implicit_optional`. | CI type-check stage. |
| NFR-MAINT-02 | Style and SQL quality | Must | Ruff and SQLFluff MUST pass for changed application and transformation code. | CI lint stage. |
| NFR-OBS-01 | Structured run logs | Must | Logs MUST include logical date, batch ID, file name when relevant, task name, and status. | Runbook demo. |
| NFR-OBS-02 | Auditability | Must | Every load attempt MUST have audit counts, status, checksum, start time, and finish time. | Integration test. |
| NFR-USE-01 | Report contract clarity | Must | JSON/CSV exports MUST use the exact view columns, explicit currency suffixes, decimal strings, stable ordering, and documented empty results. | TC-REP-001, TC-REP-002 and doc 08 snapshots. |
| NFR-USE-02 | CLI discoverability | Should | CLI help SHOULD list the six allowed views, supported filters, formats, and exit codes without requiring a console or browser. | Named verification `DEMO-CLI-HELP-01`. |
| NFR-PORT-01 | Local portability and persistence | Must | Must scope MUST run on Windows 11 with WSL2 Ubuntu, macOS, and Linux after initial downloads, offline with local fixtures. Loopback bindings, repeatable start/stop, and persistent data MUST meet doc 06. | Trainer pre-check and TC-LOCAL-001 through TC-LOCAL-004. |
| NFR-LIC-01 | Licence compliance | Must | Olist attribution and CC BY-NC-SA 4.0 terms MUST be visible in README and data dictionary. | Documentation review. |

## Hardware profiles

| Profile | CPU | RAM | Active services | Limits |
|---|---:|---:|---|---|
| Lite | 4 cores | 8 GB | PostgreSQL, Airflow 3 LocalExecutor, Mailpit; short-lived Python commands | Airflow parallelism 1, max active DAG runs 1, dbt threads 1; no student UI services. |
| Standard | 6 or more cores | 16 GB | PostgreSQL, Airflow, Mailpit; short-lived Python commands | Airflow parallelism 4, max active DAG runs 2; Should features and performance tests are easier. |

Windows users SHOULD use WSL2 memory `4GB`, swap `4GB` on an 8 GB host, or memory `8GB`, swap `4GB` on a 16 GB host. See doc 06 for proposed aggregate RAM budgets and trainer pre-check. These are design limits, not measured application results.

## Performance and scalability

ShopSight is a local training pipeline. It does not need horizontal production scaling. It MUST process synthetic Olist-equivalent historical data and deterministic daily splits; real Olist input is optional. It SHOULD meet the 30-minute full backfill and 5-minute daily run targets after the student completes basic optimisation.

## Reliability

- The pipeline MUST be idempotent for each logical date.
- Load audit records MUST distinguish `loaded`, `skipped`, `quarantined`, `reconciled`, and `failed`.
- Alerts are at-least-once. They MUST include a stable run ID for de-duplication.
- Backfills MUST be restartable without duplicate accepted rows.
- FX tasks MUST fail loudly after retry exhaustion.
- Recorded FX mode MUST be chosen before a run; live failures MUST NOT change the selected mode. Stop/start MUST preserve committed states, raw rows, marts, and audit history.

## Security

| OWASP concern | ShopSight control |
|---|---|
| Broken access control | Separate local database users and report-reader SELECT-only access. |
| Cryptographic failures | No committed passwords, API keys, or real `.env` files. |
| Injection | Validate CLI inputs and avoid string-built database commands. |
| Security misconfiguration | Pin Docker image tags and bind exposed doc 06 ports to `127.0.0.1`. |
| Vulnerable components | Run pip-audit or equivalent and Trivy in CI. |
| Logging failures | Include run IDs, logical dates, and failure reasons in logs. |

## Privacy

Default synthetic data is fictional; opt-in Olist data is public and anonymized. The student MUST still treat identifiers, city, state, and zip-code prefix with care. Under India DPDP Act 2023 principles, combined location and customer identifiers can become personal data. The project MUST NOT add real people, phone numbers, addresses, or private e-mail addresses.

## Maintainability

- Keep simulator, raw ingestion, FX ingestion, dbt, Airflow, local operations, and report-export code separate.
- Record major choices as ADRs.
- Use stable terms: logical date, batch ID, source rows, accepted rows, quarantined rows.
- Keep sample data deterministic and small.
- Keep dbt model descriptions and the data dictionary synchronized.

## Observability

The runbook MUST tell a data engineer where to find Airflow task status, task logs, `raw_load_audit`, `raw_quarantine`, dbt test failures, quality reports, and alert examples. Each log line for load and FX work MUST include the logical date.

## CLI usability

Analysts export the views for sales leaders. Help and errors MUST describe allowed views and filters. Standard output is JSON or CSV only; logs and errors go to standard error. Empty JSON results have `row_count=0` and `rows=[]`; empty CSV results contain only the exact header. No business function depends on built-in Airflow or Mailpit screens.

## Portability and licence compliance

Mandatory services MUST run locally and free of cost. Docker image tags MUST be pinned. The project MUST not depend on Bitnami images, MailHog, mandatory MinIO, or paid cloud services. Olist and Kaggle attribution MUST be part of the student documentation.

[Back to README](../README.md)
