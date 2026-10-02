# Glossary and resources

Purpose: This document defines ShopSight terms and lists official documentation and free learning resources.

## Glossary

| Term | Definition |
|---|---|
| Accepted row | A source row that passed validation and was loaded into a raw table. |
| Airflow | The orchestrator that runs ShopSight tasks in the correct order. |
| Analytics view | A query output that answers one business question from marts. |
| Audit table | A table that records load attempts, counts, checksums, status, and times. |
| Attempt ID | A unique identifier for one file-load attempt. |
| Backfill | A run that processes older logical dates again or for the first time. |
| Batch ID | A logical batch identifier shared in logs and alerts for a run. |
| Business key | A source identifier with business meaning, such as `customer_unique_id`. |
| Carried-forward rate | An FX rate copied from the last available market day. |
| Checksum | A calculated file fingerprint used to detect duplicate or changed files. |
| DAG | A directed acyclic graph of Airflow tasks. |
| Data dictionary | Documentation that defines tables, columns, grains, and meanings. |
| dbt | A transformation and testing tool for building warehouse models. |
| Dimension table | A descriptive table such as customer, product, seller, or date. |
| Fact table | A measurement table, such as orders or order items. |
| Frankfurter | The free BRL to INR API used only in opt-in live mode; recorded historical responses serve the explicit default local mode. |
| Freight | Delivery charge. ShopSight reports it separately from GMV. |
| GMV | Gross merchandise value. In ShopSight it is the sum of item prices only. |
| Grain | The meaning of one row in a table. |
| Idempotency | Safe rerun behaviour where the same logical date does not create duplicates. |
| Landing folder | A date-partitioned folder such as `landing/date=2018-01-02/`. |
| Lite profile | The four-core, 8 GB laptop setup with one LocalExecutor task/active DAG run, short-lived CLI exports, and no student UI services. |
| Logical date | The business date processed by one pipeline run. |
| Mart | A trusted analytics table or view built for reporting. |
| Olist | The Brazilian e-commerce public dataset provider used for training data. |
| Payment reconciliation | A check that payments equal item prices plus freight within ±1%. |
| Quarantine | Storage for rejected rows with rule IDs and reasons. |
| Raw layer | Database tables that store accepted source rows plus load metadata. |
| Reconciliation | A count or money check that proves pipeline totals still match. |
| SCD Type 1 | A dimension pattern that keeps only the latest attributes. |
| SCD Type 2 | A dimension pattern that stores historical versions with valid dates. |
| Source freshness | A check that source data arrived recently enough for the run. |
| Star schema | A warehouse design with facts connected to dimensions. |
| Surrogate key | A warehouse-generated key that identifies a dimension or fact row. |
| Synthetic local mode | Locally generated fictional data with the same Olist schemas, keys, and seed; the deliberate default, not a download-failure fallback. |
| Report-export CLI | Python command that reads one of six analytics views and writes exact JSON or CSV without recalculating business rules. |
| Recorded FX mode | Explicit offline ingestion of historical responses; never automatically entered after live retry failure. |
| Loopback binding | Publishing a local service only on `127.0.0.1`, not on the host's external interfaces. |
| Persistent local volume | Project-scoped storage that survives stop/start and is deleted only by explicitly confirmed reset. |
| Window function | SQL that calculates rankings or running metrics across related rows. |

## Official documentation links

| Tool | Official link |
|---|---|
| Python 3.12 | https://docs.python.org/3.12/ |
| uv | https://docs.astral.sh/uv/ |
| pandas | https://pandas.pydata.org/docs/ |
| Polars | https://docs.pola.rs/ |
| pandera | https://pandera.readthedocs.io/ |
| PostgreSQL | https://www.postgresql.org/docs/ |
| psycopg | https://www.psycopg.org/psycopg3/docs/ |
| dbt | https://docs.getdbt.com/ |
| dbt-postgres | https://docs.getdbt.com/docs/core/connect-data-platform/postgres-setup |
| Apache Airflow | https://airflow.apache.org/docs/apache-airflow/stable/ |
| Python argparse | https://docs.python.org/3.12/library/argparse.html |
| Python CSV | https://docs.python.org/3.12/library/csv.html |
| Python JSON | https://docs.python.org/3.12/library/json.html |
| Docker Engine | https://docs.docker.com/engine/ |
| Docker Compose | https://docs.docker.com/compose/ |
| Mailpit | https://mailpit.axllent.org/ |
| pytest | https://docs.pytest.org/ |
| coverage.py | https://coverage.readthedocs.io/ |
| Ruff | https://docs.astral.sh/ruff/ |
| mypy | https://mypy.readthedocs.io/ |
| SQLFluff | https://docs.sqlfluff.com/ |
| pre-commit | https://pre-commit.com/ |
| pip-audit | https://pypi.org/project/pip-audit/ |
| Trivy | https://trivy.dev/latest/docs/ |
| Gitleaks | https://github.com/gitleaks/gitleaks |
| Frankfurter | https://frankfurter.dev/ |
| Kaggle Olist dataset | https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce |
| CC BY-NC-SA 4.0 | https://creativecommons.org/licenses/by-nc-sa/4.0/ |

## Olist attribution

ShopSight uses the Olist Brazilian E-Commerce Public Dataset from Kaggle for non-commercial training. The dataset is attributed to Olist and Kaggle and is listed as CC BY-NC-SA 4.0. Students MUST keep this attribution in their README and data dictionary.

## Free learning resources

| Topic | Resource |
|---|---|
| PostgreSQL SQL | https://www.postgresql.org/docs/current/tutorial.html |
| dbt fundamentals | https://docs.getdbt.com/docs/introduction |
| Airflow concepts | https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/index.html |
| pandas user guide | https://pandas.pydata.org/docs/user_guide/index.html |
| Polars user guide | https://docs.pola.rs/user-guide/ |
| Docker getting started | https://docs.docker.com/get-started/ |
| Python command-line parsing | https://docs.python.org/3.12/library/argparse.html |
| Decimal arithmetic | https://docs.python.org/3.12/library/decimal.html |
| GitHub Actions | https://docs.github.com/en/actions |
| GitHub Flow | https://docs.github.com/en/get-started/using-github/github-flow |

## Suggested reading order

1. Read the README and docs 01 to 04 to understand scope and requirements.
2. Read doc 07 before writing ingestion or dbt models.
3. Read doc 08 before creating Airflow tasks and backfill behaviour.
4. Read doc 09 before adding tests.
5. Read this document when a term or tool is unclear.
6. Revisit doc 13 during weeks 5 and 6 for interview practice.
7. Use doc 06 for offline local operation and doc 08 for report interfaces. Resources support Python/data/backend/DevOps only; built-in Airflow/Mailpit consoles are optional operations inspection, not development exercises.

[Back to README](../README.md)
