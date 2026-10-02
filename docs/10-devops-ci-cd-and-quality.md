# DevOps, CI/CD, and quality

Purpose: This document defines ShopSight Git workflow, CI gates, quality rules, configuration, releases, and Definition of Done.

## Git workflow

ShopSight uses GitHub Flow. The `main` branch MUST always contain a working pipeline. Every feature, fix, and documentation change MUST use a pull request.

| Practice | Rule |
|---|---|
| Branch source | Start from current `main`. |
| Branch size | One feature area or one fix per branch. |
| PR size | Prefer fewer than 400 changed lines outside generated dbt docs. |
| Review | The trainer reviews at least 2 substantive PRs each week. |
| Merge | Merge only after CI is green and the checklist is complete. |
| Direct push | Direct pushes to `main` are not allowed. |

## Branch naming

| Work type | Pattern | Example |
|---|---|---|
| Feature | `feat/<fr-id>-short-name` | `feat/fr-raw-01-quarantine` |
| Fix | `fix/<area>-short-name` | `fix/fx-carried-forward-rate` |
| Tests | `test/<requirement-id>-short-name` | `test/fr-ci-01-sample-limit` |
| Docs | `docs/<topic>` | `docs/runbook-fx-failure` |
| ADR | `adr/<decision>` | `adr/report-serialization` |

## Conventional Commits

Each commit message MUST use Conventional Commits. Include the requirement ID when it helps review.

| Type | Example |
|---|---|
| `feat` | feat(raw): add quarantine audit for FR-RAW-01 |
| `fix` | fix(fx): fail after two retries for FR-FX-01 |
| `test` | test(dbt): cover payment tolerance for BR-19 |
| `docs` | docs(runbook): add missing-file recovery steps |
| `refactor` | refactor(loader): separate checksum lookup |
| `chore` | chore(deps): update uv lockfile |

## Pull request checklist

- The PR links at least one issue or requirement ID.
- CI is green on the PR branch.
- The change has tests for Must behaviour.
- Python coverage remains at least 85% line and 75% branch.
- dbt tests pass for changed models.
- SQLFluff passes for changed SQL.
- No full Olist data files are committed.
- No secrets or `.env` files are committed.
- The PR description states any significant AI assistance.
- Documentation, runbook, or ADRs are updated when behaviour changes.
- Report snapshots and local acceptance tests remain green; no frontend or cloud-deployment task is added.

## Branch protection

`main` MUST require these protections.

| Protection | Required setting |
|---|---|
| Pull request before merge | Enabled. |
| Required checks | Lint, type check, tests, coverage, dbt build, report exports, local acceptance, security scans. |
| Conversation resolution | Required before merge. |
| Stale approval dismissal | SHOULD be enabled after major changes. |
| Force pushes | Disabled. |
| Deletion | Disabled. |

## Quality tools and rules

| Tool | Rule |
|---|---|
| Ruff | Lint and format Python application, DAG, and tests. |
| mypy | `disallow_untyped_defs` and `no_implicit_optional` for application packages. |
| pytest | Run at least 50 meaningful unit, integration, DAG, reconciliation, report CLI, and local acceptance tests. |
| pytest-cov | Fail below 85% line coverage or 75% branch coverage for application, Airflow helpers, report/local CLI packages, and `shopsight.common`. Enforce line and branch gates separately. |
| pandera | Validate landing data before raw load. |
| dbt tests | Run generic, singular, and unit tests. Run source freshness as a separate CI step before `dbt build`. |
| SQLFluff | Lint dbt SQL with project rules. |
| pre-commit | Run Ruff, mypy, whitespace fixers, and Gitleaks locally. |
| pip-audit | Fail on high or critical vulnerable Python dependencies unless documented. |
| Trivy | Scan runtime images and fail on high or critical untriaged findings. |
| Gitleaks | Fail if secrets appear in the working tree or Git history. |

## CI pipeline

CI MUST run on every pull request and every push to `main`.

| Job | Trigger | Steps in words | Gate or failure condition |
|---|---|---|---|
| Metadata check | PR and `main` push | Check branch name, sample file sizes, and row limits. | Fails if any source sample has more than 1,000 rows. |
| Test inventory | PR and `main` push | Collect Python/dbt cases and map implemented cases or named reviews to the catalog. | Fails below 50 meaningful cases, 5 singular dbt tests, or 3 dbt unit tests; duplicated parameter labels alone do not satisfy the minimum. |
| Python lint | PR and `main` push | Install from `uv.lock`, run Ruff format check and lint. | Fails on lint or format differences. |
| Type check | PR and `main` push | Run mypy on simulator, loader, FX, report/local CLIs, and DAG packages. | Fails on untyped functions or optional errors. |
| Unit tests | PR and `main` push | Run pytest without external services. | Fails on any test failure. |
| PostgreSQL integration | PR and `main` push | Start PostgreSQL 18 service, load sample data, run ingestion tests. | Fails on reconciliation or idempotency mismatch. |
| dbt source freshness | PR and `main` push | Run `dbt source freshness` on sample daily raw and FX sources. | Fails when `loaded_at_utc` freshness is stale. Static bootstrap sources are exempt from row-age freshness. |
| Logical-date completeness | PR and `main` push | Count committed states for all 9 files before the build. | Fails when any file lacks a successful committed state; header-only daily files count. |
| dbt build | PR and `main` push | Build staging, intermediate, marts, tests, and docs on sample data. | Fails on model, generic, singular, or unit test failure. |
| Report contracts | PR and `main` push | Export all six fixture views as JSON/CSV; test filters, errors, restricted reader, and atomic files. | TC-REP-001 through TC-REP-005 and TC-IT-011 must pass. |
| Local acceptance | PR and `main` push | After cached locked installs and pinned image preparation, run the doc 06 lite entry points with external runtime network blocked and isolated project storage. | TC-LOCAL-001 through TC-LOCAL-004 must pass: clean start/health, offline demo, persistence, and input/dependency failures. |
| DAG integrity | PR and `main` push | Import Airflow DAGs and check owner, tags, retries, retry delay, and cycles. | Fails unless every pipeline task has exactly 2 retries and a 5-minute retry delay. |
| SQL quality | PR and `main` push | Run SQLFluff on dbt models and analytics views. | Fails on lint violations. |
| Security | PR and `main` push | Run Gitleaks, pip-audit, and Trivy. | Fails on secrets or untriaged high findings. |
| Coverage report | PR and `main` push | Publish line and branch coverage summary. | Fails below 85% line or 75% branch. |

Initial dependency downloads, hosted CI, artifact uploads, and vulnerability database updates may need internet. Runtime acceptance uses only local synthetic/recorded fixtures and local service networking. Retry tests use recorded failing responses and a virtual clock; deployed Airflow settings remain exactly 2 retries and 5-minute delay. No CI job starts, styles, exports, or tests a student frontend.

## Docker and Compose requirements

- Docker image tags MUST pin a version. Do not use `latest`.
- The official Airflow 3.3 Compose pattern MUST use LocalExecutor.
- PostgreSQL MUST use a pinned PostgreSQL 18 image.
- Mailpit MUST replace MailHog for alert testing.
- The lite profile MUST start PostgreSQL, Airflow, and Mailpit only.
- Python report exports are short-lived commands, not a service.
- Compose service names MUST make ownership clear, such as database, Airflow, Mailpit, and dbt.
- All published ports MUST bind to `127.0.0.1` using doc 06's fixed `15434`, `18080`, `11027`, and `18027` mappings.
- Implement only the doc 06 start/stop entry points; stop preserves project-scoped named volumes and bind mounts. Reset requires explicit confirmation and never deletes another project's resources.
- Initialization MUST apply versioned warehouse/Airflow migrations and fictional seed setup idempotently. Runtime tests MUST not require GitHub, Kaggle, live FX, or a built-in console.
- Container logs SHOULD be structured as JSON where the tool supports it.

## Environment variables

| Name | Example value | Purpose | Secret? |
|---|---|---|---|
| `SHOP_LOGICAL_DATE` | `2018-01-02` | Manual run date for local tasks. | No |
| `SHOP_LANDING_ROOT` | `landing` | Root folder for daily drops. | No |
| `SHOP_SIMULATOR_SEED` | `20261002` | Deterministic split and problem injection. | No |
| `SHOP_SOURCE_MODE` | `synthetic` | Explicit default local generation; `olist` is opt-in. | No |
| `SHOP_FX_MODE` | `fixture` | Explicit recorded historical source; `live` is opt-in and never falls back. | No |
| `SHOP_POSTGRES_HOST` | `127.0.0.1` | Host CLI database address; containers use the internal service name. | No |
| `SHOP_POSTGRES_PORT` | `15434` | Host CLI database port; containers use internal port `5432`. | No |
| `SHOP_POSTGRES_DB` | `shopsight` | Application database name. | No |
| `SHOP_POSTGRES_USER` | `shopsight_loader` | Loader database user. | No |
| `SHOP_POSTGRES_PASSWORD` | `change-me-local` | Local database password. | Yes |
| `AIRFLOW__CORE__EXECUTOR` | `LocalExecutor` | Airflow executor setting. | No |
| `SHOP_FX_BASE_URL` | `https://api.frankfurter.dev` | Frankfurter API host. | No |
| `SHOP_FX_FIXTURE_DIR` | `data/fixtures/fx` | Recorded historical FX fixtures for default local runtime, tests, and CI. | No |
| `SHOP_ALERT_EMAIL_TO` | `trainer@example.com` | Mailpit alert recipient for failures. | No |
| `SHOP_REPORT_READ_USER` | `report_reader` | SELECT-only access to the six analytics views. | No |
| `SHOP_REPORT_READ_PASSWORD` | `change-me-local` | Local report-reader database password. | Yes |

The student repository MUST include `.env.example` with placeholder values only. The real `.env` file MUST stay out of Git.

## Versioning and release

| Item | Rule |
|---|---|
| Versioning | Use SemVer tags such as `v0.1.0` and `v1.0.0`. |
| Release notes | Keep a CHANGELOG in the student repository. |
| Demo release | Tag the final viva version as `v1.0.0`. |
| ADRs | Record stack and scope decisions before implementation. |
| dbt docs | Regenerate docs when mart schemas change. |

## Dependency management

- Commit `uv.lock`.
- Keep Airflow and dbt dependencies isolated if conflicts appear.
- Pin Docker images to versioned tags.
- Review Dependabot PRs with CI before merge.
- Update sample data only through a reviewed PR.
- Do not change dbt engine or allowed data tools without an ADR; no frontend alternatives or cloud exercises are permitted.

## Definition of Done

A ShopSight feature is done only when these points are true.

1. The linked FR or NFR acceptance criteria are satisfied.
2. Tests prove positive, negative, and idempotency cases where relevant.
3. CI passes on the pull request.
4. The runbook or data dictionary is updated when behaviour changes.
5. Errors include logical date, batch ID, file name, or model name where relevant.
6. No secret, full dataset, or generated local volume is committed.
7. The student can explain the change in a Friday demo.
8. Local startup, exact JSON/CSV exports, stop/start persistence, and clear invalid-input/dependency failures have the applicable doc 09 evidence.

[Back to README](../README.md)
