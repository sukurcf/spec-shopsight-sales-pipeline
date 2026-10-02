# Non-functional requirements

Purpose: This document defines measurable quality targets for ShopSight.

## NFR catalog

| ID | Title | Priority | Measurable target | Verification method |
|---|---|---|---|---|
| NFR-PERF-01 | Full backfill time | Should | About 100,000 Olist orders SHOULD backfill in less than 30 minutes on the standard profile. | Timed local demo with clean volumes. |
| NFR-PERF-02 | Daily run time | Should | One logical date SHOULD finish in less than 5 minutes after setup. | Timed Airflow run on a representative date. |
| NFR-PERF-03 | CI duration | Must | Pull-request CI MUST finish in 15 minutes or less with sample data. | GitHub Actions run duration. |
| NFR-SCALE-01 | Local data volume | Must | Must scope MUST handle all Olist historical CSV rows and CI samples of at most 1,000 rows per source table. | Backfill log and CI data check. |
| NFR-REL-01 | Logical-date idempotency | Must | Running the same logical date twice with unchanged input checksums MUST keep accepted counts and mart totals identical. | Integration test. |
| NFR-REL-02 | Retry policy | Must | Airflow tasks that touch files, database, FX, dbt, or alerts MUST use 2 retries and 5-minute retry delay. | DAG integrity test. |
| NFR-REL-03 | FX failure safety | Must | If Frankfurter fails after retries, the run MUST fail and alert. It MUST NOT write a made-up rate. | Fault-injection test. |
| NFR-SEC-01 | Secret handling | Must | Secrets MUST come from environment variables and MUST NOT appear in Git history. | Gitleaks scan and PR review. |
| NFR-SEC-02 | Database least privilege | Must | Separate users SHOULD be used for raw load, transformations, dashboard read, and Airflow metadata. | Configuration review. |
| NFR-SEC-03 | Dependency scanning | Must | CI MUST run Python dependency and container vulnerability scans. | CI gate. |
| NFR-PRIV-01 | Personal-data awareness | Must | The student MUST document that customer IDs and location fields can become personal data under DPDP Act 2023 principles. | Documentation review. |
| NFR-MAINT-01 | Typed Python boundary | Must | Application packages MUST pass mypy with `disallow_untyped_defs` and `no_implicit_optional`. | CI type-check stage. |
| NFR-MAINT-02 | Style and SQL quality | Must | Ruff and SQLFluff MUST pass for changed application and transformation code. | CI lint stage. |
| NFR-OBS-01 | Structured run logs | Must | Logs MUST include logical date, batch ID, file name when relevant, task name, and status. | Runbook demo. |
| NFR-OBS-02 | Auditability | Must | Every load attempt MUST have audit counts, status, checksum, start time, and finish time. | Integration test. |
| NFR-USE-01 | Dashboard clarity | Must | Dashboard chart titles MUST use business terms and state BRL or INR for money. | Demo checklist. |
| NFR-ACC-01 | Dashboard accessibility | Should | Charts SHOULD avoid color-only meaning and use readable labels at 1366×768 resolution. | Manual review. |
| NFR-PORT-01 | Local portability | Must | Must scope MUST run on Windows 11 with WSL2, macOS, and Linux. | Trainer pre-check and student setup evidence. |
| NFR-LIC-01 | Licence compliance | Must | Olist attribution and CC BY-NC-SA 4.0 terms MUST be visible in README and data dictionary. | Documentation review. |

## Hardware profiles

| Profile | CPU | RAM | Active services | Limits |
|---|---:|---:|---|---|
| Lite | 4 cores | 8 GB | PostgreSQL, Airflow 3 LocalExecutor, Mailpit | Airflow parallelism low; dashboard runs on demand; no dashboard while Airflow is active. |
| Standard | 6 or more cores | 16 GB | PostgreSQL, Airflow, Mailpit, dashboard | Should features and performance tests are easier. |

Windows users SHOULD use these WSL2 values on an 8 GB laptop: memory `4GB`, swap `4GB`, processors `4`. On a 16 GB laptop, use memory `8GB`, swap `4GB`, processors `6`. The trainer MUST validate the lite profile on an 8 GB laptop before week 1.

## Performance and scalability

ShopSight is a local training pipeline. It does not need horizontal production scaling. It MUST process the Olist historical data and deterministic daily splits. It SHOULD meet the 30-minute full backfill and 5-minute daily run targets after the student completes basic optimisation.

## Reliability

- The pipeline MUST be idempotent for each logical date.
- Load audit records MUST distinguish `loaded`, `skipped`, `quarantined`, `reconciled`, and `failed`.
- Alerts are at-least-once. They MUST include a stable run ID for de-duplication.
- Backfills MUST be restartable without duplicate accepted rows.
- FX tasks MUST fail loudly after retry exhaustion.

## Security

| OWASP concern | ShopSight control |
|---|---|
| Broken access control | Separate local database users and dashboard read-only access. |
| Cryptographic failures | No committed passwords, API keys, or real `.env` files. |
| Injection | Validate CLI inputs and avoid string-built database commands. |
| Security misconfiguration | Pin Docker image tags and document exposed local ports. |
| Vulnerable components | Run pip-audit or equivalent and Trivy in CI. |
| Logging failures | Include run IDs, logical dates, and failure reasons in logs. |

## Privacy

The Olist data is public and anonymized. The student MUST still treat identifiers, city, state, and zip-code prefix with care. Under India DPDP Act 2023 principles, combined location and customer identifiers can become personal data. The project MUST NOT add real people, phone numbers, addresses, or private e-mail addresses.

## Maintainability

- Keep simulator, raw ingestion, FX ingestion, dbt, Airflow, and dashboard code separate.
- Record major choices as ADRs.
- Use stable terms: logical date, batch ID, source rows, accepted rows, quarantined rows.
- Keep sample data deterministic and small.
- Keep dbt model descriptions and the data dictionary synchronized.

## Observability

The runbook MUST tell a data engineer where to find Airflow task status, task logs, `raw_load_audit`, `raw_quarantine`, dbt test failures, quality reports, and alert examples. Each log line for load and FX work MUST include the logical date.

## Usability and accessibility

Dashboard users are sales leaders and analysts. Chart titles MUST use names such as monthly GMV, top categories, late delivery, seller rank, and payment mix. Charts SHOULD not rely only on red and green color. Empty states MUST be understandable.

## Portability and licence compliance

Mandatory services MUST run locally and free of cost. Docker image tags MUST be pinned. The project MUST not depend on Bitnami images, MailHog, mandatory MinIO, or paid cloud services. Olist and Kaggle attribution MUST be part of the student documentation.

[Back to README](../README.md)
