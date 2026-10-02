# Evaluation rubric

Purpose: This document defines mandatory gates, scoring criteria, bonus rules, deductions, grade bands, and viva questions for ShopSight.

## Mandatory gates

If any gate fails, the result is Rework required for any score.

| Gate | Project-specific evidence |
|---|---|
| CI is green on `main` | GitHub Actions pass lint, mypy, pytest, coverage, SQLFluff, dbt build, DAG integrity, and security scans. |
| Coverage thresholds are met | Python line coverage is at least 85%. Branch coverage is at least 75%. |
| Clean clone starts | README steps start PostgreSQL, Airflow, Mailpit, and sample pipeline without hidden local state. |
| No secrets in Git history | Gitleaks scan passes and no `.env` with real passwords is committed. |
| Student can explain code | In viva, the student explains ingestion, idempotency, dbt models, Airflow DAG, and one failing test live. |
| Licence compliance | README and data dictionary credit Olist and Kaggle and state CC BY-NC-SA 4.0. |

## Weighted rubric

| Category | Points |
|---|---:|
| Functional completeness (Must FRs work in the demo) | 30 |
| Code quality and design | 15 |
| Testing and coverage | 15 |
| DevOps: CI/CD, Docker, reproducibility | 10 |
| Documentation (README, ADRs, API/CLI/pipeline docs) | 10 |
| Git and engineering practices (PRs, commits, issues) | 5 |
| Final demo and viva | 15 |
| **Total** | **100** |

## Criteria by category

| Category | Excellent | Good | Needs work |
|---|---|---|---|
| Functional completeness | All Must FRs work. The demo shows one daily run, one 3-day backfill, quarantine, idempotency, carried-forward FX, dbt build, 6 views, and 4 charts. | All Must FRs mostly work. One non-critical demo step needs explanation, but data totals remain correct. | Any Must FR is missing, duplicate loads occur, FX silently falls back, or dashboard reads raw tables. |
| Code quality and design | Python packages are separated by simulator, ingestion, FX, orchestration, and dashboard. dbt layers have clear grains and names. Money uses decimals throughout. | Most boundaries are clear. Small helper modules are mixed, but tests cover the risk. | Large mixed scripts hide business logic, money uses floats, or `customer_id` is used as customer business key. |
| Testing and coverage | Coverage gates pass. Tests cover bad rows, idempotency, FX fixtures, dbt singular rules, dbt unit tests, DAG integrity, and sample row limit. | Coverage passes. Most critical tests exist, but one Should or edge case is manual. | Coverage fails, tests call live Frankfurter in CI, or dbt key tests are missing. |
| DevOps: CI/CD, Docker, reproducibility | Clean clone works on lite profile. CI uses PostgreSQL service and pinned images. Security scans are active. | Clean clone works after a documented manual fix. Pinned images and scans are mostly correct. | Setup depends on local hidden files, images use `latest`, or Airflow and dashboard overload the lite profile. |
| Documentation | README, runbook, data dictionary, dbt docs, ADRs, and CHANGELOG are consistent. Olist attribution is visible. | Main docs are complete. One ADR or runbook section lacks detail but does not block operation. | Runbook cannot recover failures, data dictionary misses mart grains, or licence notes are absent. |
| Git and engineering practices | Small PRs, clear issues, Conventional Commits, protected `main`, and meaningful reviews are visible. | PRs and commits are mostly clear. A few branches are large but reviewed. | Direct pushes to `main`, unclear commit messages, or no issue tracking. |
| Final demo and viva | The student explains ETL versus ELT, star schema grain, SQL windows, Airflow retries, dbt tests, idempotency, and one live change. | The student explains most choices and can trace one metric from CSV to dashboard. | The student cannot explain committed code, model grains, or why reruns are safe. |

## Bonus rules

Bonus is up to +10 points. Bonus applies only to Could items. It applies only when the base score is 60 or more and all mandatory gates pass. The final score is capped at 100. Should work is scored inside the base categories.

| Bonus item | Maximum points | Conditions |
|---|---:|---|
| Local Parquet lake with DuckDB | 4 | ADR explains query value, local resource cost, and Must gates remain green. |
| Extra data-quality tool | 2 | Great Expectations or Soda adds checks beyond pandera and dbt tests. |
| Astronomer Cosmos for dbt orchestration | 2 | ADR explains why Cosmos is useful and keeps Airflow stable on the standard profile. |
| OpenLineage or cloud warehouse stretch | 2 | Lineage is visible locally, or cloud use includes cost warning and destroy-after-demo steps. |

## Deductions

| Problem | Deduction |
|---|---:|
| Full Olist dataset committed to Git | Up to 10 points and possible rework. |
| Olist attribution missing from README or data dictionary | Up to 10 points and gate failure if unresolved. |
| Live FX network calls in CI | Up to 8 points. |
| Dashboard has unreadable money labels or no empty state | Up to 5 points. |
| Friday demos missed without notice | Up to 5 points. |
| AI assistance not disclosed in PR description | Up to 5 points. |
| Inconsistent requirement IDs in docs or tests | Up to 4 points. |

## Grade bands

| Score | Grade |
|---:|---|
| 85-100 | Distinction |
| 70-84 | Merit |
| 60-69 | Pass |
| Below 60 | Rework required |

## Project-specific viva questions

1. Why is `customer_unique_id` the customer business key, and why is `customer_id` not enough?
2. How does ShopSight prove `source rows = accepted rows + quarantined rows` for one file?
3. What happens when the same logical date and checksum are loaded twice?
4. How do you convert BRL to INR for a weekend date with no direct rate?
5. Why does GMV exclude freight, and where is freight reported?
6. What is the grain of `fct_order_items`, and which columns prove it?
7. Which two analytics views use window functions, and why are windows useful there?
8. What does Airflow retry twice, and what happens after the final FX failure?
9. How do dbt generic tests differ from singular tests in your project?
10. How does the lite profile protect an 8 GB laptop during the demo?

[Back to README](../README.md)
