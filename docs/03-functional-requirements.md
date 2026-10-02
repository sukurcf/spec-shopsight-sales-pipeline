# Functional requirements

Purpose: This document defines ShopSight behaviour for simulation, ingestion, FX, transformations, quality, orchestration, analytics, dashboard, CI, documentation, and licence compliance.

## Requirement summary

| ID | Title | Priority | Roles | Linked BRs |
|---|---|---|---|---|
| FR-SIM-01 | Create deterministic daily Olist drops | Must | Data engineer | BR-01, BR-02, BR-03, BR-21 |
| FR-RAW-01 | Validate, load, audit, quarantine, and reconcile raw files | Must | Data engineer | BR-02, BR-03, BR-04, BR-05, BR-06 |
| FR-RAW-02 | Enforce idempotency for file loads and logical dates | Must | Data engineer | BR-07, BR-08 |
| FR-FX-01 | Ingest BRL to INR exchange rates | Must | Data engineer | BR-09, BR-10, BR-11, BR-12 |
| FR-DBT-01 | Build dbt layers and star schema | Must | Analytics engineer | BR-13, BR-14, BR-15 |
| FR-DBT-02 | Apply money and payment reconciliation rules | Must | Analytics engineer, Data analyst | BR-16, BR-17, BR-18, BR-19 |
| FR-DQ-01 | Run required data-quality tests | Must | Analytics engineer | BR-20 |
| FR-ORCH-01 | Orchestrate the daily and backfill pipeline | Must | Data engineer | BR-08, BR-11, BR-22 |
| FR-ANA-01 | Publish 6 required analytics views | Must | Data analyst | BR-13, BR-16, BR-17, BR-23 |
| FR-DASH-01 | Provide a 4-chart sales dashboard | Must | Head of Sales, Data analyst | BR-24, BR-25 |
| FR-CI-01 | Run CI with PostgreSQL and sample data | Must | Data engineer, Trainer | BR-20, BR-26 |
| FR-DOC-01 | Publish dbt docs, data dictionary, and runbook | Must | Data engineer, Analytics engineer | BR-27 |
| FR-LIC-01 | Document Olist licence, attribution, and fallback data | Must | Trainer, Data engineer | BR-21, BR-28 |
| FR-SIM-02 | Configure simulator problem rates and late updates | Should | Data engineer | BR-03, BR-08 |
| FR-DBT-03 | Add customer SCD Type 2 | Should | Analytics engineer | BR-14 |
| FR-ANA-02 | Publish review-delay and retention analytics | Should | Data analyst | BR-23 |
| FR-DQ-02 | Publish run quality report and performance evidence | Should | Data engineer | BR-20, BR-22 |
| FR-EXT-01 | Add local Parquet lake or extra quality tool | Could | Data engineer | BR-29 |
| FR-EXT-02 | Add lineage or cloud warehouse stretch | Could | Data engineer | BR-29 |

## Must requirements

### FR-SIM-01 — Create deterministic daily Olist drops

The simulator MUST split the historical Olist CSV data by `order_purchase_timestamp` into `landing/date=YYYY-MM-DD/` folders. It MUST use a documented seed value of `20261002` for the default split. It MUST create all 9 source files for each logical date, even when a file has only its header for that day.

| Field | Value |
|---|---|
| Priority | Must |
| Roles | Data engineer |
| Linked BRs | BR-01, BR-02, BR-03, BR-21 |

Acceptance criteria:

1. Given the full Olist dataset and seed `20261002`, when the simulator builds `landing/date=2018-01-02/`, then that folder contains exactly the 9 documented CSV file names.
2. Given two simulator runs with seed `20261002`, when row counts are compared for `olist_orders_dataset.csv`, then every logical date has the same count in both runs.
3. Given a missing source file named `olist_products_dataset.csv`, when the simulator starts, then it stops before writing a partial date folder and reports `SIM-MISSING-SOURCE`.
4. Given Kaggle data is unavailable, when synthetic fallback is selected, then output files keep the same 9 file names and column names.

### FR-RAW-01 — Validate, load, audit, quarantine, and reconcile raw files

The raw loader MUST read one logical date folder and validate file names, column names, required values, types, and key formats. Accepted rows MUST load into PostgreSQL `raw` tables with `attempt_id`, `batch_id`, `logical_date`, `file_name`, `file_checksum`, and `loaded_at_utc`. Rejected rows MUST load into `raw_quarantine` with a rule ID, reason, file name, and source row number.

| Field | Value |
|---|---|
| Priority | Must |
| Roles | Data engineer |
| Linked BRs | BR-02, BR-03, BR-04, BR-05, BR-06 |

Acceptance criteria:

1. Given `landing/date=2018-01-02/` has 9 valid files, when the loader runs with `batch_id=20180102T000000Z`, then each accepted raw row stores that batch ID and its source file name.
2. Given one `olist_orders_dataset.csv` row has blank `order_id`, when validation runs, then that row is written to `raw_quarantine` with rule ID `VAL-ORDER-ID-REQUIRED`.
3. Given 100 source rows and 3 rejected rows for one file, when load audit is written, then `source_count=100`, `accepted_count=97`, and `quarantined_count=3`.
4. Given audit counts do not satisfy `source rows = accepted rows + quarantined rows`, when reconciliation runs, then the load status becomes `failed` and marts are not built.

### FR-RAW-02 — Enforce idempotency for file loads and logical dates

The loader MUST prevent duplicate accepted rows for the current committed file state. Each load attempt MUST get a separate attempt ID. A file MUST be skipped only when its checksum equals the current committed checksum for that logical date and file name. A rerun with changed file content MUST create a new attempt and replace only affected purchase-date partitions after validation succeeds.

| Field | Value |
|---|---|
| Priority | Must |
| Roles | Data engineer |
| Linked BRs | BR-07, BR-08 |

Acceptance criteria:

1. Given `olist_order_items_dataset.csv` for `2018-01-02` is committed with checksum `abc123`, when the same file is loaded again, then the new attempt status is `skipped` and accepted row count does not increase.
2. Given the same logical date has changed checksum `def456`, when validation succeeds, then the committed file state changes to `def456` and facts rebuild only affected purchase-date partitions.
3. Given a rerun fails during validation, when the previous successful rows exist, then the previous accepted rows remain available and the failed attempt is recorded separately.
4. Given the committed file changes from checksum `abc123` to `def456` and back to `abc123`, when both correction loads succeed, then the committed rows again match the original `abc123` source.

### FR-FX-01 — Ingest BRL to INR exchange rates

The FX task MUST fetch BRL to INR rates from the official Frankfurter v2 API with ECB provider rates. For a single date it MAY use `/v2/rate/brl/inr?date=YYYY-MM-DD&providers=ecb`. For a range it MUST use `/v2/rates?base=brl&quotes=inr&from=YYYY-MM-DD&to=YYYY-MM-DD&providers=ecb`. Weekends and holidays MUST use the last available rate and set `is_carried_forward=true`.

| Field | Value |
|---|---|
| Priority | Must |
| Roles | Data engineer |
| Linked BRs | BR-09, BR-10, BR-11, BR-12 |

Acceptance criteria:

1. Given logical date `2018-01-02`, when the FX task requests a single-day rate, then it stores base `BRL`, quote `INR`, provider `ecb`, `rate_date=2018-01-02`, and `is_carried_forward=false`.
2. Given date range `2018-01-01` to `2018-01-07`, when the FX task backfills rates, then it uses the Frankfurter time-series endpoint with `from=2018-01-01` and `to=2018-01-07`.
3. Given `2018-01-01` has no market-day rate and `2017-12-29` has a rate, when the FX task stores `2018-01-01`, then `is_carried_forward=true` and `source_rate_date=2017-12-29`.
4. Given Frankfurter returns HTTP `422` for an invalid currency, when 2 retries are exhausted, then the task fails, sends an alert, and does not insert a fallback rate.

### FR-DBT-01 — Build dbt layers and star schema

The dbt project MUST build staging, intermediate, and mart layers. Marts MUST include `fct_order_items`, `fct_orders`, `fct_order_payments`, `dim_customer`, `dim_product`, `dim_seller`, and `dim_date`. Facts MUST be incremental after the first historical backfill.

| Field | Value |
|---|---|
| Priority | Must |
| Roles | Analytics engineer |
| Linked BRs | BR-13, BR-14, BR-15 |

Acceptance criteria:

1. Given raw Olist tables exist, when dbt builds staging models, then source timestamps are typed as timestamps and money fields are typed as decimals.
2. Given product categories exist in Portuguese, when `dim_product` builds, then it exposes `product_category_name_english` from `product_category_name_translation.csv`.
3. Given `customer_unique_id=CUST-001` has two source `customer_id` values, when `dim_customer` builds in Must scope, then it creates one current customer row for `CUST-001`.
4. Given a fact model lacks a documented grain key, when dbt tests run, then the build fails before dashboard refresh.

### FR-DBT-02 — Apply money and payment reconciliation rules

Marts MUST use decimal money values. GMV MUST equal item price totals. Freight MUST be reported separately. Each order MUST be classified as matched, within tolerance, or failed against the ±1% payment rule. Failed orders MUST be recorded in `dq_payment_exceptions` and excluded from certified financial views. Mart-level money outputs MUST round to 2 decimals only at the final output.

| Field | Value |
|---|---|
| Priority | Must |
| Roles | Analytics engineer, Data analyst |
| Linked BRs | BR-16, BR-17, BR-18, BR-19 |

Acceptance criteria:

1. Given an order has item prices `100.00` and `50.00`, when `fct_orders` builds, then `gmv_brl=150.00` and freight is not included in GMV.
2. Given the same order has freight values `10.00` and `5.00`, when `fct_orders` builds, then `freight_brl=15.00` in a separate column.
3. Given payment total is `166.00` and item plus freight total is `165.00`, when reconciliation runs, then status is `within_tolerance` because the difference is within ±1%.
4. Given payment total is `170.00` and item plus freight total is `165.00`, when reconciliation runs, then the order is recorded in `dq_payment_exceptions` and excluded from certified financial views.
5. Given mart output rounds money, when values `12.345` and `12.355` are formatted, then half-up rounding returns `12.35` and `12.36`.

### FR-DQ-01 — Run required data-quality tests

The project MUST run Python validation before raw load and dbt tests after transformations. dbt generic tests MUST cover all mart primary keys and foreign keys. At least 5 singular tests and at least 3 dbt unit tests MUST exist for business rules and complex models.

| Field | Value |
|---|---|
| Priority | Must |
| Roles | Analytics engineer |
| Linked BRs | BR-20 |

Acceptance criteria:

1. Given all staging, intermediate, and mart grain keys, when dbt tests run, then each key has both `unique` and `not_null` tests.
2. Given all fact-to-dimension keys, when dbt tests run, then each foreign key has a relationship test.
3. Given singular tests run, then at least 5 tests cover delivered date order, non-negative money, payment exception exclusion, carried-forward FX, and source-to-mart revenue totals.
4. Given CI runs without network access, when FX tests run, then they use recorded fixture files and do not call Frankfurter.

### FR-ORCH-01 — Orchestrate the daily and backfill pipeline

Airflow 3.x MUST orchestrate a daily DAG with LocalExecutor. The order MUST be wait for landing files, validate and load raw, fetch FX rates, run dbt source freshness, run dbt build, publish quality summary, and notify. Tasks MUST use 2 retries, a 5-minute retry delay, timeouts, and failure alerts.

| Field | Value |
|---|---|
| Priority | Must |
| Roles | Data engineer |
| Linked BRs | BR-08, BR-11, BR-22 |

Acceptance criteria:

1. Given all 9 files exist for `2018-01-02`, when the DAG runs, then tasks execute in the documented dependency order.
2. Given `product_category_name_translation.csv` is missing, when the sensor reaches its timeout, then the run fails with the missing file name in the alert.
3. Given the FX task fails twice and then fails a third time, when retries are exhausted, then the DAG sends a Mailpit e-mail or webhook message with a stable run ID.
4. Given a backfill from `2018-01-01` to `2018-01-03`, when the command completes, then exactly 3 logical dates have completed audit records.

### FR-ANA-01 — Publish 6 required analytics views

The marts MUST expose 6 analytics views. At least 2 views MUST use SQL window functions. Views MUST answer monthly GMV, top 10 categories, delivery time versus estimate by state, late-delivery percentage, seller ranking, and payment-method mix.

| Field | Value |
|---|---|
| Priority | Must |
| Roles | Data analyst |
| Linked BRs | BR-13, BR-16, BR-17, BR-23 |

Acceptance criteria:

1. Given delivered orders in January 2018, when `mart_monthly_gmv` is queried, then it returns `month`, `order_count`, `item_count`, `gmv_brl`, `gmv_inr`, and `freight_brl`.
2. Given category GMV for a selected month, when `mart_top_categories` is queried, then it returns at most 10 categories ordered by `gmv_brl` descending.
3. Given seller monthly GMV, when `mart_seller_ranking` is queried, then it includes a window rank column named `seller_rank`.
4. Given an analytics view reads a raw table directly, when review runs, then the view is rejected because analytics must read marts.

### FR-DASH-01 — Provide a 4-chart sales dashboard

The dashboard MUST show at least 4 charts for monthly GMV, top categories, late delivery by state, and payment-method mix. Streamlit or Metabase OSS is allowed. In the 8 GB lite profile, the dashboard MUST run on demand and not at the same time as Airflow.

| Field | Value |
|---|---|
| Priority | Must |
| Roles | Head of Sales, Data analyst |
| Linked BRs | BR-24, BR-25 |

Acceptance criteria:

1. Given marts are built, when the Head of Sales opens the dashboard, then 4 charts show labels with BRL or INR where money appears.
2. Given the lite profile is active, when Airflow is running, then the documented run procedure keeps the dashboard stopped.
3. Given a chart source view is empty, when the dashboard opens, then it shows `No data for selected date range` instead of a stack trace.

### FR-CI-01 — Run CI with PostgreSQL and sample data

GitHub Actions MUST run on pull requests and pushes to `main`. CI MUST use a PostgreSQL service container and deterministic sample data with at most 1,000 rows per source table. CI MUST run lint, type checks, pytest, coverage, SQLFluff, DAG integrity checks, and dbt build.

| Field | Value |
|---|---|
| Priority | Must |
| Roles | Data engineer, Trainer |
| Linked BRs | BR-20, BR-26 |

Acceptance criteria:

1. Given a pull request changes ingestion code, when CI runs, then Ruff, mypy, pytest, coverage, and PostgreSQL integration tests complete.
2. Given sample `olist_orders_dataset.csv` has 1,001 data rows, when CI data checks run, then CI fails with `CI-SAMPLE-ROW-LIMIT`.
3. Given Python line coverage is 84%, when CI evaluates coverage, then the build fails because the threshold is 85%.
4. Given a live HTTP request is attempted during CI FX tests, when network blocking is active, then the test fails and reports that fixtures are required.

### FR-DOC-01 — Publish dbt docs, data dictionary, and runbook

The student repository MUST include dbt documentation with lineage, a data dictionary, and a pipeline runbook. The runbook MUST cover daily run, backfill, missing files, quarantine review, FX failure, dbt test failure, and dashboard refresh.

| Field | Value |
|---|---|
| Priority | Must |
| Roles | Data engineer, Analytics engineer |
| Linked BRs | BR-27 |

Acceptance criteria:

1. Given the final repository, when the trainer opens the data dictionary, then it defines every mart table and analytics view.
2. Given an FX failure alert, when the runbook is followed, then it names where to find retries, fixture status, and the failed logical date.
3. Given dbt docs are generated, when lineage is reviewed, then `fct_order_items` shows upstream raw or staging dependencies.

### FR-LIC-01 — Document Olist licence, attribution, and fallback data

The project MUST document that the Olist Brazilian E-Commerce Public Dataset comes from Kaggle and uses CC BY-NC-SA 4.0. The student MUST credit Olist and Kaggle. The synthetic fallback MUST use the same file names, column names, and key relationships.

| Field | Value |
|---|---|
| Priority | Must |
| Roles | Trainer, Data engineer |
| Linked BRs | BR-21, BR-28 |

Acceptance criteria:

1. Given the student README, when licence notes are reviewed, then it states Olist, Kaggle, CC BY-NC-SA 4.0, and non-commercial training use.
2. Given fallback synthetic data is used, when the loader validates files, then all 9 files match the real Olist column names.
3. Given the dashboard is demoed, when the trainer asks for source attribution, then the student can point to the README and data dictionary.

## Should and Could requirements

| ID | Description | Acceptance criteria |
|---|---|---|
| FR-SIM-02 | The simulator SHOULD allow configured duplicate, null, bad-date, bad-money, and late-status-update rates. | Given `duplicate_rate=0.02`, one seeded run creates the same duplicate row set each time. Given `late_update_rate=0.01`, only existing order IDs are updated. |
| FR-DBT-03 | `dim_customer` SHOULD support SCD Type 2 on city and state. | Given one `customer_unique_id` moves state on `2018-06-01`, facts before that date reference the old version and facts after it reference the new version. |
| FR-ANA-02 | The marts SHOULD expose review-score-versus-delay and repeat-customer retention views. | Given repeat buyers exist, the retention view returns `cohort_month`, `age_month`, `customers`, and `repeat_customer_rate`. |
| FR-DQ-02 | Each run SHOULD publish a Markdown or HTML quality report and SHOULD record performance evidence. | Given a daily run completes, the report shows source, accepted, quarantined, dbt failures, and elapsed time. |
| FR-EXT-01 | The project MAY add a local Parquet lake or Great Expectations or Soda after Must work is complete. | Given all Must CI gates pass, the ADR explains added value and local resource cost. |
| FR-EXT-02 | The project MAY add OpenLineage or a free-tier cloud warehouse after Must work is complete. | Given cloud is used, the ADR includes cost warning and destroy-after-demo steps. |

## Business rules

| BR ID | Exact rule | Exact values | Used by |
|---|---|---|---|
| BR-01 | Logical dates use `YYYY-MM-DD`. Machine event timestamps use UTC. | Example logical date `2018-01-02`; UTC suffix `Z`. | FR-SIM-01 |
| BR-02 | Daily landing folders use one fixed path format. | `landing/date=YYYY-MM-DD/` | FR-SIM-01, FR-RAW-01 |
| BR-03 | A complete daily drop contains the 9 real Olist CSV file names. | `olist_customers_dataset.csv`, `olist_geolocation_dataset.csv`, `olist_order_items_dataset.csv`, `olist_order_payments_dataset.csv`, `olist_order_reviews_dataset.csv`, `olist_orders_dataset.csv`, `olist_products_dataset.csv`, `olist_sellers_dataset.csv`, `product_category_name_translation.csv` | FR-SIM-01, FR-RAW-01, FR-SIM-02 |
| BR-04 | Raw accepted rows keep load metadata. | `attempt_id`, `batch_id`, `logical_date`, `file_name`, `file_checksum`, `loaded_at_utc`. | FR-RAW-01 |
| BR-05 | Quarantine rows keep rejection context. | Rule ID, reason, file name, source row number, raw payload. | FR-RAW-01 |
| BR-06 | Reconciliation is mandatory for every loaded file. | `source_count = accepted_count + quarantined_count`. | FR-RAW-01 |
| BR-07 | Duplicate-file checks use the current committed file state. | Skip only when logical date, file name, and checksum match the current committed state. Failed attempts never commit. | FR-RAW-02 |
| BR-08 | Idempotency is defined per logical date and current input. | Unchanged reruns keep counts and totals unchanged. Corrected inputs rebuild affected purchase-date partitions and remove missing fact keys. | FR-RAW-02, FR-ORCH-01 |
| BR-09 | FX base and quote are fixed for board revenue. | Base `BRL`; quote `INR`. | FR-FX-01 |
| BR-10 | Frankfurter v2 endpoint formats are fixed. | Single date `/v2/rate/brl/inr?date=YYYY-MM-DD&providers=ecb`; range `/v2/rates?base=brl&quotes=inr&from=YYYY-MM-DD&to=YYYY-MM-DD&providers=ecb`. | FR-FX-01 |
| BR-11 | FX failures do not silently fallback after retries fail. | 2 retries; failure alert after final failure. | FR-FX-01, FR-ORCH-01 |
| BR-12 | Closed-day rates must be flagged. | `is_carried_forward=true`; keep `source_rate_date`. | FR-FX-01 |
| BR-13 | Star schema mart names are fixed. | `fct_order_items`, `fct_orders`, `fct_order_payments`, `dim_customer`, `dim_product`, `dim_seller`, `dim_date`. | FR-DBT-01, FR-ANA-01 |
| BR-14 | Customer business key is fixed. | Use `customer_unique_id`; use `customer_id` only for source joins. | FR-DBT-01, FR-DBT-03 |
| BR-15 | Fact grains are fixed. | One row per order item; one row per order; one row per order payment sequence. | FR-DBT-01 |
| BR-16 | GMV excludes freight. | GMV is sum of `price`; freight is sum of `freight_value`. | FR-DBT-02, FR-ANA-01 |
| BR-17 | Money uses decimal types. | No floating point money values. | FR-DBT-02, FR-ANA-01 |
| BR-18 | Rounding happens only at mart output level. | Half-up rounding to 2 decimal places; examples `12.345` to `12.35` and `12.355` to `12.36`. | FR-DBT-02 |
| BR-19 | Payment reconciliation tolerance is fixed. | Orders outside ±1% go to `dq_payment_exceptions` and stay out of certified financial views. | FR-DBT-02 |
| BR-20 | Test minimums are fixed. | 5 singular dbt tests; 3 dbt unit tests; 85% line and 75% branch coverage. | FR-DQ-01, FR-CI-01, FR-DQ-02 |
| BR-21 | Dataset licence and fallback rules are fixed. | Olist Kaggle data; CC BY-NC-SA 4.0; synthetic fallback same schema. | FR-SIM-01, FR-LIC-01 |
| BR-22 | Airflow retry values are fixed. | 2 retries; 5-minute retry delay. | FR-ORCH-01, FR-DQ-02 |
| BR-23 | Required analytics count is fixed. | 6 Must views; at least 2 with window functions. | FR-ANA-01, FR-ANA-02 |
| BR-24 | Dashboard Must scope count is fixed. | At least 4 charts. | FR-DASH-01 |
| BR-25 | Lite profile constraint is fixed. | 8 GB RAM; dashboard on demand; not active with Airflow. | FR-DASH-01 |
| BR-26 | CI sample size is fixed. | At most 1,000 rows per source table. | FR-CI-01 |
| BR-27 | Documentation deliverables are fixed. | dbt docs, data dictionary, pipeline runbook. | FR-DOC-01 |
| BR-28 | Attribution must be visible. | Credit Olist and Kaggle in README and data dictionary. | FR-LIC-01 |
| BR-29 | Stretch work starts only after Must scope passes. | All Must CI gates green before Could work. | FR-EXT-01, FR-EXT-02 |

## State machines

### Raw file load state

```mermaid
stateDiagram-v2
    [*] --> Discovered
    Discovered --> Skipped
    Discovered --> Validating
    Validating --> Loaded
    Validating --> Quarantined
    Loaded --> Reconciled
    Reconciled --> Published
    Discovered --> Missing
    Validating --> Failed
    Loaded --> Failed
    Failed --> Retrying
    Retrying --> Validating
```

| From | Event | Guard | To | Actor |
|---|---|---|---|---|
| Discovered | Duplicate detected | Incoming checksum equals current committed checksum | Skipped | Raw loader |
| Discovered | Start validation | 9 files found | Validating | Airflow |
| Validating | Valid rows loaded | Rejected rows stored | Loaded | Raw loader |
| Validating | All rows rejected | Quarantine written | Quarantined | Raw loader |
| Loaded | Count check passes | Source equals accepted plus quarantined | Reconciled | Raw loader |
| Reconciled | dbt starts | Reconciliation complete | Published | Airflow |
| Discovered | Sensor timeout | Any file missing | Missing | Airflow |
| Validating | Schema error | Required column missing | Failed | Raw loader |
| Loaded | Count mismatch | Reconciliation false | Failed | Raw loader |
| Failed | Retry available | Retry count less than 2 | Retrying | Airflow |

### FX rate state

```mermaid
stateDiagram-v2
    [*] --> Requested
    Requested --> DirectRate
    Requested --> CarryForward
    Requested --> RetryWaiting
    RetryWaiting --> Requested
    RetryWaiting --> Failed
    DirectRate --> Stored
    CarryForward --> Stored
```

| From | Event | Guard | To | Actor |
|---|---|---|---|---|
| Requested | API returns dated BRL INR rate | Rate date equals logical date | DirectRate | FX task |
| Requested | API has no closed-day row | Earlier rate exists | CarryForward | FX task |
| Requested | HTTP 5xx or timeout | Retry count less than 2 | RetryWaiting | Airflow |
| RetryWaiting | Delay elapsed | 5 minutes passed | Requested | Airflow |
| RetryWaiting | Final retry failed | Retry count equals 2 | Failed | Airflow |
| DirectRate | Store row | Valid decimal rate | Stored | FX task |
| CarryForward | Store row | `source_rate_date` present | Stored | FX task |

## Validation rules

| Field | Rule | Error message or code |
|---|---|---|
| `order_id` | Required in orders, items, payments, and reviews. | `VAL-ORDER-ID-REQUIRED` |
| `customer_id` | Required in `olist_orders_dataset.csv` and customers source. | `VAL-CUSTOMER-ID-REQUIRED` |
| `customer_unique_id` | Required in customers source and used as business key. | `VAL-CUSTOMER-UNIQUE-REQUIRED` |
| `order_item_id` | Positive integer within an order. | `VAL-ORDER-ITEM-ID-POSITIVE` |
| `price` | Decimal greater than or equal to `0.00`. | `VAL-PRICE-NONNEGATIVE` |
| `freight_value` | Decimal greater than or equal to `0.00`. | `VAL-FREIGHT-NONNEGATIVE` |
| `payment_value` | Decimal greater than or equal to `0.00`. | `VAL-PAYMENT-NONNEGATIVE` |
| `order_purchase_timestamp` | Valid timestamp and present for every order. | `VAL-PURCHASE-TIMESTAMP` |
| `order_delivered_customer_date` | For delivered orders, not earlier than purchase timestamp. | `VAL-DELIVERY-BEFORE-PURCHASE` |
| `order_status` | One of documented Olist statuses. | `VAL-ORDER-STATUS` |
| `payment_type` | One of documented Olist payment types. | `VAL-PAYMENT-TYPE` |
| Source file header | Exact real Olist column names in documented order. | `VAL-SOURCE-SCHEMA` |

## Error scenarios

| Condition | System response | User-visible message or status |
|---|---|---|
| Landing folder missing | Sensor waits until timeout, then fails run. | `Missing landing/date=YYYY-MM-DD/` |
| One CSV file missing | Load does not start for that date. | `Missing required file: product_category_name_translation.csv` |
| Wrong column name | File is rejected before accepted rows load. | `VAL-SOURCE-SCHEMA` |
| Duplicate checksum | Loader records skipped audit status. | `Skipped already loaded file` |
| Bad row value | Row is stored in `raw_quarantine`. | Rule ID such as `VAL-PRICE-NONNEGATIVE` |
| Reconciliation mismatch | Load status becomes failed. | `RAW-RECONCILIATION-FAILED` |
| Frankfurter unavailable | Task retries twice and alerts after final failure. | `FX fetch failed after 2 retries` |
| dbt test failure | Publish is blocked. | Failing model or test name |
| Dashboard view empty | Dashboard shows empty-state message. | `No data for selected date range` |
| CI sample too large | CI fails before dbt build. | `CI-SAMPLE-ROW-LIMIT` |

[Back to README](../README.md)
