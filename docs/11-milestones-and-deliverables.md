# Milestones and deliverables

Purpose: This document gives the six-week ShopSight plan, effort budget, final checklist, submission process, and demo script.

## Effort budget

The total budget is about 180 hours. Must work is about 120 hours, including about 30 hours of learning and setup. Should, Could, hardening, interview practice, and contingency use the remaining 60 hours.

| Feature area | Requirement IDs | Must hours | Later hours | Notes |
|---|---|---:|---:|---|
| Learning, setup, Git workflow | FR-CI-01 | 30 | 4 | Covers Docker, PostgreSQL, dbt, Airflow, and CI basics. |
| Daily-drop simulator and sample data | FR-SIM-01, FR-SIM-02 | 10 | 5 | Seed `20261002`, 9 files, and problem injection. |
| Raw ingestion, validation, quarantine, audit | FR-RAW-01, FR-RAW-02 | 18 | 4 | Includes reconciliation and rerun safety. |
| Exchange-rate ingestion | FR-FX-01 | 8 | 2 | Uses Frankfurter fixtures in tests. |
| dbt warehouse and marts | FR-DBT-01, FR-DBT-02, FR-DBT-03 | 20 | 10 | Must uses Type 1 customer. Should adds Type 2. |
| Data quality and tests | FR-DQ-01, FR-DQ-02 | 10 | 8 | Includes 5 singular tests and 3 dbt unit tests. |
| Airflow orchestration and alerts | FR-ORCH-01 | 10 | 4 | Daily DAG, backfill, retries, Mailpit or webhook. |
| Analytics views and dashboard | FR-ANA-01, FR-DASH-01, FR-ANA-02 | 8 | 8 | Six views, four Must charts, two Should views. |
| CI, security, docs, runbook, licence | FR-CI-01, FR-DOC-01, FR-LIC-01 | 6 | 10 | Includes Olist and Kaggle attribution. |
| Performance, polish, viva, contingency | NFR-PERF-01, NFR-PERF-02 | 0 | 5 | Time boxed after Must scope passes. |
| **Total** |  | **120** | **60** | **180 hours** |

## Week-by-week plan

| Week | Goals | Tasks with IDs | Deliverables | Friday demo checkpoint |
|---|---|---|---|---|
| 1 | Set up tools and design the first slice. | Create `shopsight`, branch protection, Docker services, sample data plan, ADRs for dashboard and data frame tool. Read FR-SIM-01, FR-RAW-01, FR-CI-01. | Repository skeleton, CI skeleton, `.env.example`, first ADRs, tiny sample folder. | Show PostgreSQL and Mailpit running, plus one CI check. |
| 2 | Build deterministic input and raw loading. | Implement seeded daily folders for FR-SIM-01. Validate 9 files for FR-RAW-01. Add quarantine and audit counts. | Simulator, raw tables, quarantine examples, unit tests. | Show `landing/date=2018-01-02/` and one quarantined bad row. |
| 3 | Add idempotency, FX, and early dbt layers. | Finish FR-RAW-02. Add Frankfurter fixtures for FR-FX-01. Create staging and intermediate models for FR-DBT-01. | Rerun test, FX fixture tests, staging docs. | Run the same logical date twice with unchanged input checksums and prove counts do not change. |
| 4 | Complete Must pipeline, dashboard, and required documentation. | Finish facts, dimensions, money rules, required dbt tests, Airflow daily DAG, 6 views, 4 charts, dbt docs, data dictionary, runbook, and licence notes. Cover FR-DBT-02, FR-DQ-01, FR-ORCH-01, FR-ANA-01, FR-DASH-01, FR-DOC-01, and FR-LIC-01. | End-to-end sample run, dbt build, DAG integrity tests, dashboard, runbook, data dictionary, licence notes. | Show daily DAG order, dbt tests, the four charts, and Olist attribution. |
| 5 | Harden and add selected Should items. | Add Markdown or HTML quality report, performance evidence, SCD Type 2 if chosen, extra analytics, and accessibility fixes. Cover FR-DQ-02, FR-DBT-03, FR-ANA-02 where selected. | Quality report, timed run logs, updated ADRs, polished docs. | Show one failure recovery and performance evidence. |
| 6 | Prepare final submission and viva. | Polish runbook, data dictionary, README, demo video, final tag, and mock interview. Re-verify FR-DOC-01 and FR-LIC-01. | Final `v1.0.0` tag, clean CI, demo video, viva notes. | Deliver 10-minute final demo rehearsal. |

## Gantt chart

```mermaid
gantt
    title ShopSight six-week plan
    dateFormat  YYYY-MM-DD
    section Week 1
    Setup and learning           :a1, 2026-10-05, 5d
    Repository and CI skeleton   :a2, 2026-10-07, 3d
    section Week 2
    Simulator and sample data    :b1, 2026-10-12, 3d
    Raw validation and audit     :b2, 2026-10-14, 3d
    section Week 3
    Idempotency and FX fixtures  :c1, 2026-10-19, 3d
    dbt staging and intermediate :c2, 2026-10-21, 3d
    section Week 4
    Marts and data quality       :d1, 2026-10-26, 3d
    Airflow and dashboard        :d2, 2026-10-28, 3d
    section Week 5
    Should scope and hardening   :e1, 2026-11-02, 5d
    section Week 6
    Documentation and viva       :f1, 2026-11-09, 5d
```

## Final deliverables checklist

| Deliverable | Required evidence |
|---|---|
| Public GitHub repository `shopsight` | Trainer `@sukurcf` is a collaborator. |
| README | Clean-clone setup in 10 steps or fewer, Olist attribution, and screenshots or chart descriptions. |
| Architecture diagram | Shows landing, raw, dbt, marts, Airflow, dashboard, and Mailpit. |
| ADRs | At least 5 ADRs, including dashboard, data frame engine, dbt v2 if used, SCD Type 2 if used, and alerting. |
| Test and coverage report | 85% line and 75% branch coverage, plus dbt test output. |
| Airflow evidence | DAG graph, successful daily run, backfill example, and failed-alert example. |
| dbt docs | Generated lineage and model descriptions. |
| Data dictionary | Tables, grains, keys, money rules, and Olist attribution. |
| Pipeline runbook | Daily run, backfill, missing files, quarantine, FX failure, dbt failure, dashboard refresh. |
| Demo video | Maximum 5 minutes. Shows setup, run, quality, and dashboard. |
| Final tag | `v1.0.0` on the submitted commit. |

## Submission process

1. Confirm CI is green on `main`.
2. Confirm no secrets are present in Git history.
3. Tag the final commit as `v1.0.0`.
4. Add the demo video link to the student README.
5. Open a final GitHub Issue titled `[Submission] ShopSight final review`.
6. Include the repository link, tag, CI run link, demo video link, and known limitations.
7. Do not change the submission after the trainer starts evaluation unless asked.

## 10-minute final demo script

| Minute | What to show | Requirements covered |
|---:|---|---|
| 0-1 | State the problem, Olist source, CC BY-NC-SA 4.0 attribution, and BRL to INR need. | FR-LIC-01 |
| 1-2 | Show repository layout, ADRs, `.env.example`, and CI status. | FR-CI-01, FR-DOC-01 |
| 2-3 | Show `landing/date=2018-01-02/` with 9 files and the seed `20261002`. | FR-SIM-01 |
| 3-4 | Run or show raw load evidence with audit counts and one quarantine row. | FR-RAW-01 |
| 4-5 | Rerun the same logical date and show skipped checksum or unchanged totals for unchanged input. | FR-RAW-02 |
| 5-6 | Show FX fixture use, carried-forward flag, and failure alert example. | FR-FX-01, FR-ORCH-01 |
| 6-7 | Show dbt lineage, facts, dimensions, tests, and money reconciliation. | FR-DBT-01, FR-DBT-02, FR-DQ-01 |
| 7-8 | Show Airflow DAG order and a 3-day backfill audit. | FR-ORCH-01 |
| 8-9 | Show 6 analytics views and 4 dashboard charts with INR labels. | FR-ANA-01, FR-DASH-01 |
| 9-10 | Show runbook and data dictionary, then explain one defect, its fix, and audit evidence. | FR-DOC-01, NFR-MAINT-01, NFR-OBS-02 |

[Back to README](../README.md)
