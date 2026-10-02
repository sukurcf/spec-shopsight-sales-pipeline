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
| ADR | `adr/<decision>` | `adr/dashboard-streamlit` |

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

## Branch protection

`main` MUST require these protections.

| Protection | Required setting |
|---|---|
| Pull request before merge | Enabled. |
| Required checks | Lint, type check, tests, coverage, dbt build, security scans. |
| Conversation resolution | Required before merge. |
| Stale approval dismissal | SHOULD be enabled after major changes. |
| Force pushes | Disabled. |
| Deletion | Disabled. |

## Quality tools and rules

| Tool | Rule |
|---|---|
| Ruff | Lint and format Python application, DAG, and tests. |
| mypy | `disallow_untyped_defs` and `no_implicit_optional` for application packages. |
| pytest | Run unit, integration, DAG integrity, and reconciliation tests. |
| pytest-cov | Fail below 85% line coverage or 75% branch coverage for application, Airflow helpers, dashboard helpers, and `shopsight.common`. |
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
| Python lint | PR and `main` push | Install from `uv.lock`, run Ruff format check and lint. | Fails on lint or format differences. |
| Type check | PR and `main` push | Run mypy on simulator, loader, FX, dashboard, and DAG packages. | Fails on untyped functions or optional errors. |
| Unit tests | PR and `main` push | Run pytest without external services. | Fails on any test failure. |
| PostgreSQL integration | PR and `main` push | Start PostgreSQL 18 service, load sample data, run ingestion tests. | Fails on reconciliation or idempotency mismatch. |
| dbt source freshness | PR and `main` push | Run `dbt source freshness` on sample daily raw and FX sources. | Fails when `loaded_at_utc` freshness is stale. Static bootstrap sources are exempt from row-age freshness. |
| Logical-date completeness | PR and `main` push | Count committed states for all 9 files before the build. | Fails when any file lacks a successful committed state; header-only daily files count. |
| dbt build | PR and `main` push | Build staging, intermediate, marts, tests, and docs on sample data. | Fails on model, generic, singular, or unit test failure. |
| DAG integrity | PR and `main` push | Import Airflow DAGs and check owner, tags, retries, retry delay, and cycles. | Fails unless every pipeline task has exactly 2 retries and a 5-minute retry delay. |
| SQL quality | PR and `main` push | Run SQLFluff on dbt models and analytics views. | Fails on lint violations. |
| Security | PR and `main` push | Run Gitleaks, pip-audit, and Trivy. | Fails on secrets or untriaged high findings. |
| Coverage report | PR and `main` push | Publish line and branch coverage summary. | Fails below 85% line or 75% branch. |

## Docker and Compose requirements

- Docker image tags MUST pin a version. Do not use `latest`.
- The official Airflow 3.3 Compose pattern MUST use LocalExecutor.
- PostgreSQL MUST use a pinned PostgreSQL 18 image.
- Mailpit MUST replace MailHog for alert testing.
- The lite profile MUST start PostgreSQL, Airflow, and Mailpit only.
- The dashboard MUST start on demand in the lite profile.
- Compose service names MUST make ownership clear, such as database, Airflow, Mailpit, dbt, and dashboard.
- Container logs SHOULD be structured as JSON where the tool supports it.

## Environment variables

| Name | Example value | Purpose | Secret? |
|---|---|---|---|
| `SHOP_LOGICAL_DATE` | `2018-01-02` | Manual run date for local tasks. | No |
| `SHOP_LANDING_ROOT` | `landing` | Root folder for daily drops. | No |
| `SHOP_SIMULATOR_SEED` | `20261002` | Deterministic split and problem injection. | No |
| `SHOP_POSTGRES_HOST` | `localhost` | Application database host. | No |
| `SHOP_POSTGRES_PORT` | `5432` | Application database port. | No |
| `SHOP_POSTGRES_DB` | `shopsight` | Application database name. | No |
| `SHOP_POSTGRES_USER` | `shopsight_loader` | Loader database user. | No |
| `SHOP_POSTGRES_PASSWORD` | `change-me-local` | Local database password. | Yes |
| `AIRFLOW__CORE__EXECUTOR` | `LocalExecutor` | Airflow executor setting. | No |
| `SHOP_FX_BASE_URL` | `https://api.frankfurter.dev` | Frankfurter API host. | No |
| `SHOP_FX_FIXTURE_DIR` | `data/fixtures/fx` | Recorded FX fixtures for tests and CI. | No |
| `SHOP_ALERT_EMAIL_TO` | `trainer@example.com` | Mailpit alert recipient for failures. | No |
| `SHOP_DASHBOARD_READ_USER` | `dashboard_reader` | Read-only dashboard database user. | No |
| `SHOP_DASHBOARD_READ_PASSWORD` | `change-me-local` | Dashboard database password. | Yes |

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
- Do not change dbt v2 or dashboard tool without an ADR.

## Definition of Done

A ShopSight feature is done only when these points are true.

1. The linked FR or NFR acceptance criteria are satisfied.
2. Tests prove positive, negative, and idempotency cases where relevant.
3. CI passes on the pull request.
4. The runbook or data dictionary is updated when behaviour changes.
5. Errors include logical date, batch ID, file name, or model name where relevant.
6. No secret, full dataset, or generated local volume is committed.
7. The student can explain the change in a Friday demo.

[Back to README](../README.md)
