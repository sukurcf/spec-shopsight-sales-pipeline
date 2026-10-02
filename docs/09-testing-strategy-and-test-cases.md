# Testing strategy and test cases

Purpose: This document defines the ShopSight test approach, tools, fixtures, coverage gates, catalog, traceability, performance checks, and release criteria.

## Test strategy

ShopSight uses a data-pipeline test pyramid. Fast Python and dbt tests run first. PostgreSQL, Airflow, and dashboard checks run after the core rules pass.

| Level | Approximate count | Examples | CI gate |
|---|---:|---|---|
| Unit | 18 | Schema rules, checksums, FX carry-forward, money calculations | Yes |
| Integration | 13 | PostgreSQL raw loads, reconciliation, idempotent reruns | Yes |
| Data quality | 19 | dbt generic, singular, unit, source freshness, and analytics-result tests | Yes |
| DAG and CI | 9 | DAG import, cycles, retries, sample limits, alerts, and backfill tests | Yes |
| UI and security | 4 | Dashboard labels, empty state, secrets, read-only user | Yes or demo |
| Performance | 3 | Daily run, backfill, CI duration | Yes for CI, demo for local timing |

## Tools and reference versions

| Tool | Version line | Use |
|---|---|---|
| Python | 3.12.x | Application and tests |
| uv | 0.12.x | Dependency and lockfile management |
| pytest | 9.1.x | Python unit and integration tests |
| pytest-cov and coverage | 7.1.x and 7.16.x | Line and branch coverage |
| pandera | 0.33.x | DataFrame schema validation |
| psycopg | 3.3.x | PostgreSQL integration tests |
| PostgreSQL | 18.x | Local warehouse and CI service |
| dbt Core and dbt-postgres | 1.12.x and 1.11.x | Transformations and dbt tests |
| SQLFluff | 4.3.x | SQL style checks |
| Apache Airflow | 3.3.x | DAG integrity tests |
| Ruff | 0.16.x | Python lint and format |
| mypy | 2.4.x | Typed Python boundary |
| Streamlit or Metabase OSS | 1.64.x or 0.63.x | Dashboard checks |
| Mailpit | 1.31.x | Alert verification |
| pip-audit and Trivy | 2.10.x and 0.75.x | Dependency and image scans |

## Test environments

| Environment | Purpose | Data | Network |
|---|---|---|---|
| Developer laptop lite | Daily development on 8 GB RAM | Fixture and selected historical dates | Frankfurter allowed only outside CI |
| Developer laptop standard | Full backfill and Should performance evidence | Full Olist dataset | Frankfurter allowed for manual run |
| GitHub Actions CI | Mandatory pull-request gate | Sample fixture at or below 1,000 rows per source table | Live FX network blocked |
| Trainer demo | Final verification | Full data or fixture fallback | Alerts shown in Mailpit or webhook log |

## Test data strategy

The student MUST create deterministic fixtures. The fixture `fixture_2018_01_02_small` is the minimum CI dataset.

| Fixture item | Exact value |
|---|---|
| Logical date | `2018-01-02` |
| Seed | `20261002` |
| Required files | 9 Olist file names |
| Total source rows | 63 |
| Accepted rows after validation | 60 |
| Quarantined rows | 3 |
| Orders source rows | 8 |
| Valid accepted orders | 7 |
| Order item source rows | 10 |
| Valid accepted order items | 9 |
| Payment source rows | 8 |
| Valid accepted payments | 7 |
| Expected mart orders | 7 |
| Expected mart order items | 9 |
| Expected delivered orders | 7 |
| Expected total GMV | `900.00` BRL |
| Expected total freight | `120.00` BRL |
| Expected payment total | `1020.50` BRL |
| FX rate for 2018-01-02 | `19.50` INR per BRL |
| Expected total GMV INR | `17550.00` INR |
| Known bad rows | Blank `order_id` at data row 8; negative `price`; invalid `payment_type` |

FX tests MUST use recorded Frankfurter fixture files. The fixture for `2018-01-06` MUST carry forward the `2018-01-05` rate and set `is_carried_forward=true`.

## Coverage thresholds

| Scope | Measured packages | Line threshold | Branch threshold | Exclusions |
|---|---|---:|---:|---|
| Python application | `shopsight.simulator`, `shopsight.ingestion`, `shopsight.fx`, `shopsight.quality` | 85% | 75% | Tests, generated files, notebooks, `__main__` blocks |
| Airflow helper modules | Python functions imported by DAGs | 85% | 75% | Airflow provider internals |
| Dashboard helpers | Dashboard query and formatting helpers | 85% | 75% | Visual framework internals |
| Shared package | `shopsight.common` | 85% | 75% | Generated files and `__main__` blocks |

The CI build MUST fail when line or branch coverage is below the threshold. Line and branch gates are enforced separately.

## Test case catalog

| ID | Title | Type | Priority | Linked requirement IDs | Preconditions | Steps | Test data | Expected result |
|---|---|---|---|---|---|---|---|---|
| TC-UT-001 | Validate complete landing folder names | Unit | Must | FR-SIM-01, BR-02, BR-03 | Fixture root is empty. | Generate fixture for `2018-01-02`; list the date folder. | Seed `20261002`. | Folder `landing/date=2018-01-02/` contains exactly the 9 required file names. |
| TC-UT-002 | Repeat simulator seed is deterministic | Unit | Must | FR-SIM-01, BR-01 | Two clean output roots exist. | Generate the same date range twice; compare order-row counts by date. | Seed `20261002`; range `2018-01-01` to `2018-01-03`. | Both runs return counts `2018-01-01=5`, `2018-01-02=8`, `2018-01-03=4`. |
| TC-UT-003 | Missing source file stops simulator | Unit | Must | FR-SIM-01, BR-03 | `olist_products_dataset.csv` is absent. | Start simulation for `2018-01-02`. | Source directory without products file. | Job status is `FAILED`, error identifier is `SIM-MISSING-SOURCE`, and no date folder is created. |
| TC-UT-004 | Synthetic fallback preserves schema | Unit | Must | FR-LIC-01, BR-21 | Kaggle files are unavailable. | Generate synthetic fallback; read headers. | Fallback mode `synthetic`. | All 9 files exist and `olist_orders_dataset.csv` has the 8 documented order columns. |
| TC-UT-005 | Header mismatch fails whole file | Unit | Must | FR-RAW-01 | Raw loader is configured. | Validate orders file with `order_identifier` instead of `order_id`. | One malformed header file. | File status is `failed`; error identifier is `VAL-SOURCE-SCHEMA`; accepted count is `0`. |
| TC-UT-006 | Blank order ID is quarantined | Unit | Must | FR-RAW-01, BR-05 | Orders schema is loaded. | Validate one row with blank `order_id`. | Source row number `42`. | One quarantine row has `rule_id=VAL-ORDER-ID-REQUIRED` and `source_row_number=42`. |
| TC-UT-007 | Negative item price is quarantined | Unit | Must | FR-RAW-01 | Items schema is loaded. | Validate an item row with `price=-1.00`. | Bad price row has `order_id=ORD-BAD-PRICE`. | Row is rejected with `rule_id=VAL-PRICE-NONNEGATIVE`; accepted count is `0`. |
| TC-UT-008 | Invalid payment type is quarantined | Unit | Must | FR-RAW-01 | Payments schema is loaded. | Validate payment type `cash`. | `payment_type=cash`. | Row is rejected with `rule_id=VAL-PAYMENT-TYPE`; reason contains `accepted values`. |
| TC-UT-009 | Checksum duplicate is skipped | Unit | Must | FR-RAW-02, BR-07 | Current committed file state contains checksum `abc123`. | Evaluate load decision for same logical date, file name, and checksum. | `2018-01-02`, `olist_orders_dataset.csv`, `abc123`. | Decision is `skip` before any raw write; new audit status is `skipped`; accepted delta is `0`. |
| TC-UT-010 | Changed checksum replaces one date | Unit | Must | FR-RAW-02, BR-08 | Previous accepted rows exist. | Evaluate rerun with checksum `def456` after validation succeeds. | Same logical date, changed file checksum. | Replacement plan rebuilds affected purchase-date partitions and keeps one row per source grain. |
| TC-UT-011 | Failed rerun preserves old rows | Unit | Must | FR-RAW-02, NFR-REL-01 | Previous successful rows exist. | Validate changed file with bad header. | Existing checksum `abc123`; new checksum `bad999`. | Failed attempt is recorded, and previous accepted count remains `7` orders. |
| TC-UT-012 | FX direct market-day rate | Unit | Must | FR-FX-01, BR-09, BR-10 | FX fixture exists. | Parse fixture for `2018-01-02`. | Rate `19.50`. | Stored row has base `BRL`, quote `INR`, rate date `2018-01-02`, and `is_carried_forward=false`. |
| TC-UT-013 | FX carry-forward weekend | Unit | Must | FR-FX-01, BR-12 | FX fixture exists. | Build rates for `2018-01-06`. | Previous rate date `2018-01-05`, rate `19.5678`. | Stored row has `source_rate_date=2018-01-05` and `is_carried_forward=true`. |
| TC-UT-014 | FX retry exhaustion has no fallback | Unit | Must | FR-FX-01 plus BR-11 and NFR-REL-03 | FX client receives HTTP `422`. | Simulate initial call plus 2 retries. | Invalid currency response fixture. | Task status is `failed`; alert identifier is `FX-HTTP-RETRY-EXHAUSTED`; no rate row is written. |
| TC-UT-015 | GMV excludes freight | Unit | Must | FR-DBT-02, BR-16 | Decimal total calculator is available. | Calculate order totals. | Prices `100.00`, `50.00`; freight `10.00`, `5.00`. | `gmv_brl=150.00` and `freight_brl=15.00`. |
| TC-UT-016 | Payment within tolerance passes | Unit | Must | FR-DBT-02, BR-19 | Payment reconciliation function is available. | Reconcile payment against item plus freight total. | Payment `166.00`; expected `165.00`. | Reconciliation status is `within_tolerance`. |
| TC-UT-017 | Payment outside tolerance creates exception | Unit | Must | FR-DBT-02, BR-19 | Payment reconciliation function is available with failure reporting. | Reconcile payment against item plus freight total. | Payment `170.00`; expected `165.00`. | Reconciliation status is `failed`, and an exception record has difference `5.00`. |
| TC-UT-018 | Mart rounding only at output | Unit | Must | FR-DBT-02, BR-17, BR-18 | Money helper uses decimals. | Format mart output for values `12.345` and `12.355`. | Decimal input values. | Half-up rounding returns `12.35` and `12.36`. |
| TC-IT-001 | Load full fixture into raw | Integration | Must | FR-RAW-01, BR-04, BR-06 | PostgreSQL 18 service is running. | Load `fixture_2018_01_02_small`. | 63 source rows. | Audit total shows `source_count=63`, `accepted_count=60`, `quarantined_count=3`, status `reconciled`. |
| TC-IT-002 | Quarantine keeps raw payload | Integration | Must | FR-RAW-01, BR-05 | Fixture load completed. | Query quarantine for blank order ID row. | Source data row number `8`, excluding the header. | Row contains file name `olist_orders_dataset.csv`, rule `VAL-ORDER-ID-REQUIRED`, source row number `8`, and original blank `order_id`. |
| TC-IT-003 | Same logical date twice is idempotent | Integration | Must | FR-RAW-02, NFR-REL-01 | Fixture load completed once. | Run the same logical date a second time. | Small fixture named `fixture_2018_01_02_small`. | Accepted rows remain `60`; fct orders remain `7`; GMV remains `900.00`. |
| TC-IT-004 | Reconciliation mismatch blocks marts | Integration | Must | FR-RAW-01, BR-06 | Loader can simulate audit mismatch. | Force accepted count `59` for 63 source and 3 quarantine rows. | Mismatch fixture. | Status is `failed`, error identifier is `RAW-RECONCILIATION-FAILED`, and dbt build does not start. |
| TC-IT-005 | Changed checksum reloads one date | Integration | Must | FR-RAW-02, BR-08 | Date `2018-01-02` is loaded. | Change one valid order status and rerun after validation. | New checksum `def456`. | Exactly 7 accepted orders exist for `2018-01-02`; other dates keep their previous counts. |
| TC-IT-014 | A to B to A correction restores committed state | Integration | Must | FR-RAW-02, BR-08 | Checksum `abc123` is committed. | Load corrected checksum `def456`, then reload original checksum `abc123`. | Orders fixture versions A, B, and A. | The final committed checksum is `abc123`, and row counts and totals match the first A load. |
| TC-IT-015 | Corrected file removes missing item | Integration | Must | FR-RAW-02, BR-08 | Fixture facts are built. | Remove one item priced `40.00` from a corrected item file and rerun. | New checksum `items-minus-one`. | The removed fact row disappears, and GMV drops from `900.00` to `860.00` BRL. |
| TC-IT-006 | FX range backfill stores seven dates | Integration | Must | FR-FX-01, BR-10 | Recorded range fixture exists. | Load FX range `2018-01-01` to `2018-01-07`. | BRL to INR fixture. | Seven rate dates exist; `2018-01-06` and `2018-01-07` are carried forward. |
| TC-IT-007 | Type 1 customer uses business key | Integration | Must | FR-DBT-01, BR-14 | Raw customers include two `customer_id` values for `CUST-001`. | Build `dim_customer` in Must scope. | Two source rows for one unique customer. | `dim_customer` has one current row for `customer_unique_id=CUST-001`. |
| TC-IT-008 | Product dimension uses English category | Integration | Must | FR-DBT-01 | Raw product and translation tables exist. | Build `dim_product`. | Category `beleza_saude` maps to `health_beauty`. | `dim_product` exposes `product_category_name_english=health_beauty`. |
| TC-IT-009 | Facts have required grains | Integration | Must | FR-DBT-01, BR-13, BR-15 | dbt marts are built. | Count fact keys in fixture mart. | 7 orders and 9 items. | `fct_orders` has 7 unique `order_id` values; `fct_order_items` has 9 unique order-item keys. |
| TC-IT-010 | Revenue totals match within 0.01 percent | Integration | Must | FR-DBT-02 | Staging and mart models are built. | Compare staging GMV with mart GMV. | Expected `900.00` BRL. | Difference is `0.00%`, which is within `0.01%`. |
| TC-IT-011 | Dashboard read-only user cannot write | Integration | Must | NFR-SEC-02, FR-DASH-01 | Dashboard database user exists. | Attempt to insert into `fct_orders` as dashboard user. | Read-only connection. | Database rejects write and dashboard user can still select from `mart_monthly_gmv`. |
| TC-IT-012 | Structured logs contain run context | Integration | Must | NFR-OBS-01 | One fixture run completed. | Inspect application log events for load task. | Logical date `2018-01-02`. | Each load log event has `logical_date`, `batch_id`, `file_name`, `task_name`, and `status`. |
| TC-IT-013 | Audit row has timing and checksum | Integration | Must | NFR-OBS-02 | Fixture run completed. | Query audit row for orders file. | Orders source file `olist_orders_dataset.csv`. | Row has non-empty checksum, `started_at_utc`, `finished_at_utc`, counts, and terminal status. |
| TC-DQ-001 | dbt grain keys are tested | Data quality | Must | FR-DQ-01 | dbt project is configured. | Run dbt generic tests for staging, intermediate, and mart grain keys. | All dbt models. | Every model grain key has `unique` and `not_null`; failures count is `0`. |
| TC-DQ-002 | dbt foreign keys have relationships | Data quality | Must | FR-DQ-01 | Marts are built. | Run relationship tests from facts to dimensions. | Fixture marts. | All fact-to-dimension relationship tests pass with `0` failures. |
| TC-DQ-003 | Singular delivered-date test fails bad order | Data quality | Must | FR-DQ-01 | Singular tests exist. | Insert a delivered date before purchase date in fixture copy; run dbt tests. | Bad order `ORD-EARLY`. | Singular test fails and reports exactly 1 offending order. |
| TC-DQ-004 | Singular non-negative money test | Data quality | Must | FR-DQ-01 | Singular tests exist. | Run non-negative money singular test. | Fixture marts. | Test passes with `0` rows where money is negative. |
| TC-DQ-005 | Singular payment exception exclusion test | Data quality | Must | FR-DQ-01, BR-20 | Singular tests exist. | Run payment exception exclusion test. | Order with payment `170.00` and expected `165.00`. | Order appears in `dq_payment_exceptions` and does not appear in certified financial views. |
| TC-DQ-006 | Singular carried-forward FX test | Data quality | Must | FR-DQ-01 | FX marts are built. | Run carried-forward FX singular test. | `2018-01-06` fixture. | Test passes because source date is `2018-01-05` and flag is true. |
| TC-DQ-007 | dbt unit test order money model | Data quality | Must | FR-DQ-01, FR-DBT-02 | dbt test fixtures include order items and payments. | Execute the dbt money-model unit test for `int_order_money`. | One order with 2 items and 2 freight rows. | Unit test expects `items_total_brl=150.00`, `freight_total_brl=15.00`, `payment_total_brl=166.00`. |
| TC-DQ-008 | dbt unit test customer Type 1 | Data quality | Must | FR-DQ-01, FR-DBT-01 | dbt test fixtures include duplicate customer business keys. | Execute the dbt customer-dimension unit test for `dim_customer`. | Two source customer IDs for `CUST-001`. | Unit test expects one output row for `CUST-001`. |
| TC-DQ-009 | dbt unit test seller ranking | Data quality | Must | FR-DQ-01, FR-ANA-01 | dbt test fixtures include two monthly seller totals. | Execute the dbt seller-ranking unit test for `mart_seller_ranking`. | Sellers `S1=300.00`, `S2=200.00`. | `S1` has `seller_rank=1`; `S2` has `seller_rank=2`. |
| TC-DQ-010 | Source freshness uses loaded_at_utc | Data quality | Must | FR-DQ-01 | dbt source freshness is configured. | Run `dbt source freshness` for daily raw and FX sources. | Raw rows have `loaded_at_utc` older than 30 hours. | Freshness status is `error` because `loaded_at_utc` exceeds the error threshold; static bootstrap sources are exempt. |
| TC-DQ-013 | Logical-date completeness checks all files | Data quality | Must | FR-DQ-01 | Committed file-state table exists. | Check completeness for `2018-01-02`. | 8 files committed and 1 missing committed state. | Completeness fails and names the missing file; header-only bootstrap files count as complete. |
| TC-DQ-014 | Source-to-mart revenue singular test | Data quality | Must | FR-DQ-01, FR-DBT-02 | Singular tests exist. | Compare eligible staging and certified mart GMV. | Payment exception order is excluded from both sides. | Difference is within `0.01%`, and no exception order appears in certified output. |
| TC-DQ-015 | Top categories exact fixture result | Data quality | Must | FR-ANA-01 | Marts are built. | Read the January 2018 `health_beauty` row from `mart_top_categories`. | Fixture marts. | Rank 1 is `health_beauty`, item count is `3`, and GMV is `300.00` BRL. |
| TC-DQ-016 | Delivery state exact fixture result | Data quality | Must | FR-ANA-01 | Marts are built. | Query `mart_delivery_state` for state `SP`. | Fixture marts. | Delivered order count is `4`, average delivery days is `5.00`, and late order count is `1`. |
| TC-DQ-017 | Late delivery exact fixture result | Data quality | Must | FR-ANA-01 | Marts are built. | Sum January 2018 state-level rows from `mart_late_delivery`. | Fixture marts. | Summed delivered count is `7`, summed late count is `2`, and late delivery percent is `28.57`. |
| TC-DQ-018 | Payment mix exact fixture result | Data quality | Must | FR-ANA-01 | Marts are built. | Query `mart_payment_mix` for `credit_card`. | Fixture marts. | Payment order count is `5`, payment value is `780.50` BRL, and payment share is `76.48`. |
| TC-DQ-019 | Seller ranking exact fixture view result | Data quality | Must | FR-ANA-01 | Marts are built. | Read the `S1` seller row from `mart_seller_ranking`. | Fixture marts. | Seller `S1` has rank `1`, item count `4`, and GMV `300.00` BRL. |
| TC-DQ-011 | Analytics views read marts only | Data quality | Must | FR-ANA-01 | Static model review is enabled. | Check dependencies of the 6 analytics views. | dbt manifest. | No analytics view depends directly on `raw_olist_*` tables. |
| TC-DQ-012 | Monthly GMV view output columns | Data quality | Must | FR-ANA-01 | Marts are built. | Read the `2018-01` row from `mart_monthly_gmv`. | Fixture marts. | One row has `month=2018-01`, `order_count=7`, `item_count=9`, `gmv_brl=900.00`, `gmv_inr=17550.00`, and `freight_brl=120.00`. |
| TC-INFRA-001 | DAG imports without errors | Infra | Must | FR-ORCH-01 | Airflow 3.3.x test container is running. | Import all DAG files. | Airflow 3.3.x. | `shopsight_daily_pipeline` imports with 0 errors and owner `data-engineering`. |
| TC-INFRA-002 | DAG has no dependency cycle | Infra | Must | FR-ORCH-01 | DAG is imported. | Validate topological task order. | DAG task graph. | Order is sensor, raw load, FX, `run_dbt_source_freshness`, dbt build, quality summary, notify. |
| TC-INFRA-003 | DAG retries and tags are set | Infra | Must | FR-ORCH-01 plus NFR-REL-02 and BR-22 | DAG is imported. | Inspect task retry values and tags. | DAG metadata. | Each pipeline task has 2 retries, 5-minute delay, and DAG tags include `shopsight`, `batch`, `dbt`, `postgres`. |
| TC-INFRA-004 | CI sample row limit is enforced | Infra | Must | FR-CI-01, BR-26 | CI data check exists. | Run check on orders sample with 1,001 rows. | Oversized sample file. | Check fails with `CI-SAMPLE-ROW-LIMIT` before dbt build. |
| TC-INFRA-005 | CI blocks live FX network | Infra | Must | FR-CI-01, FR-FX-01 | CI network block is active. | Run FX tests that try a live HTTP call. | Network-block fixture. | Test fails with message `FX fixtures are required in CI`. |
| TC-INFRA-006 | Sensor timeout alerts missing file | Infra | Must | FR-ORCH-01 | Landing folder for `2018-01-02` lacks one required file. | Run the sensor with one missing file. | Missing `product_category_name_translation.csv`. | Task fails after the timeout and alert names the missing file. |
| TC-INFRA-007 | Exhausted FX retries alert | Infra | Must | FR-ORCH-01, FR-FX-01 | FX task uses recorded failing response fixtures. | Run FX task with failing response fixtures. | Three failed calls including 2 retries. | Alert has code `FX-HTTP-RETRY-EXHAUSTED` and no fallback rate is stored. |
| TC-INFRA-008 | dbt failure sends alert | Infra | Must | FR-ORCH-01 | DAG run reaches dbt after raw and FX tasks pass. | Run dbt with one failing singular test. | Bad delivered-date fixture. | DAG stops before notify success and sends a failure alert with code `DBT-TEST-FAILED`. |
| TC-INFRA-009 | Inclusive 3-day backfill completes | Infra | Must | FR-ORCH-01 | Landing folders exist for three dates. | Backfill `2018-01-01` to `2018-01-03`. | Three complete logical-date folders. | Completed audit records exist for all three dates, including both endpoints. |
| TC-SEC-001 | Secrets are not committed | Security | Must | NFR-SEC-01 | Repository has sample config only. | Run Gitleaks scan. | `.env.example` with placeholder values. | Scan reports 0 secrets and no real password value. |
| TC-SEC-002 | Dependency scan gate fails on known issue | Security | Must | NFR-SEC-03 | CI security stage is configured. | Run dependency scan against a deliberately vulnerable test lockfile copy. | Test-only vulnerable package fixture. | Security stage fails and reports the vulnerable package name. |
| TC-SEC-003 | DPDP note exists in documentation | Security | Must | NFR-PRIV-01, FR-LIC-01 | Student docs are written. | Review README and data dictionary text. | Documentation draft. | Both documents mention customer IDs, location fields, and DPDP Act 2023 awareness. |
| TC-UI-001 | Dashboard charts use business labels | UI | Must | FR-DASH-01, NFR-USE-01 | Marts are built. | Open dashboard home page. | Fixture marts. | Four charts are visible and money chart titles include `BRL` or `INR`. |
| TC-UI-002 | Dashboard empty state is friendly | UI | Must | FR-DASH-01 | Dashboard filter is available. | Select a date range with no mart rows. | Date range `2019-01-01` to `2019-01-31`. | Dashboard shows `No data for selected date range` and no stack trace. |
| TC-UI-003 | Lite profile does not run dashboard with Airflow | UI | Must | FR-DASH-01, BR-25 | Lite profile is selected. | Start Airflow; follow dashboard run procedure. | 8 GB profile. | Runbook keeps dashboard stopped until Airflow is stopped. |
| TC-PERF-001 | Daily run meets Should target | Performance | Should | NFR-PERF-02 | Standard fixture environment is warm. | Time one DAG run for `2018-01-02`. | 63-row fixture. | Run completes in less than 5 minutes and writes a quality summary. |
| TC-PERF-002 | Full backfill meets Should target | Performance | Should | NFR-PERF-01 | Standard 16 GB profile is ready. | Time full historical backfill. | About 100,000 Olist orders. | Backfill completes in less than 30 minutes or records optimization notes. |
| TC-PERF-003 | CI duration meets Must target | Performance | Must | NFR-PERF-03, FR-CI-01 | Pull request CI is configured. | Run full CI on fixture data. | Each source table has at most 1,000 rows. | CI completes in 15 minutes or less. |
| TC-CI-001 | Python coverage gates fail below line threshold | CI | Must | FR-CI-01, BR-20 | Coverage gate is configured. | Run coverage with line result `84%`. | Coverage report fixture. | CI fails because line threshold is `85%`. |
| TC-CI-002 | Branch coverage gate is separate | CI | Must | FR-CI-01, BR-20 | Coverage gate is configured. | Run coverage with line `90%` and branch `74%`. | Coverage report fixture. | CI fails because branch threshold is `75%`. |
| TC-CI-003 | Ruff mypy and SQLFluff gates run | CI | Must | FR-CI-01 with NFR-MAINT-01 and NFR-MAINT-02 | CI workflow exists. | Open one pull request with Python and dbt changes. | Fixture branch. | CI reports separate Ruff, mypy, and SQLFluff stages. |
| TC-DOC-001 | Runbook covers failure operations | Documentation | Must | FR-DOC-01, BR-27 | Student runbook is drafted. | Review runbook sections. | Runbook document. | It covers daily run, backfill, missing files, quarantine review, FX failure, dbt failure, and dashboard refresh. |
| TC-DOC-002 | Olist licence attribution is visible | Documentation | Must | FR-LIC-01 with BR-28 and NFR-LIC-01 | Student README and data dictionary exist. | Review attribution sections. | Documentation draft. | Both documents state Olist, Kaggle, CC BY-NC-SA 4.0, and non-commercial training use. |

## Traceability matrix

| Requirement or rule | Evidence |
|---|---|
| FR-SIM-01 | Simulator evidence: TC-UT-001, TC-UT-002, TC-UT-003 |
| FR-RAW-01 | Raw-load evidence: TC-UT-005, TC-UT-006, TC-IT-001, TC-IT-004 |
| FR-RAW-02 | TC-UT-009, TC-UT-010, TC-UT-011, TC-IT-003, TC-IT-005 |
| FR-FX-01 | TC-UT-012, TC-UT-013, TC-UT-014, TC-IT-006, TC-INFRA-005 |
| FR-DBT-01 | dbt schema evidence: TC-IT-007, TC-IT-008, TC-IT-009, TC-DQ-008 |
| FR-DBT-02 | TC-UT-015, TC-UT-016, TC-UT-017, TC-UT-018, TC-IT-010, TC-DQ-007 |
| FR-DQ-01 | TC-DQ-001 through TC-DQ-019 |
| FR-ORCH-01 | DAG evidence: TC-INFRA-001, TC-INFRA-002, TC-INFRA-003, TC-INFRA-006, TC-INFRA-008, TC-INFRA-009 |
| FR-ANA-01 | Analytics-view evidence: TC-DQ-009, TC-DQ-011, TC-DQ-012, TC-DQ-015, TC-DQ-016, TC-DQ-017, TC-DQ-018, TC-DQ-019 |
| FR-DASH-01 | Dashboard evidence: TC-IT-011, TC-UI-001, TC-UI-002, TC-UI-003 |
| FR-CI-01 | TC-INFRA-004, TC-INFRA-005, TC-PERF-003, TC-CI-001, TC-CI-002, TC-CI-003 |
| FR-DOC-01 | TC-DOC-001 |
| FR-LIC-01 | Licence evidence: TC-UT-004, TC-SEC-003, TC-DOC-002 |
| BR-01 | TC-UT-002 |
| BR-02 | TC-UT-001 |
| BR-03 | TC-UT-001, TC-UT-003 |
| BR-04 | TC-IT-001 |
| BR-05 | TC-UT-006, TC-IT-002 |
| BR-06 | TC-IT-001, TC-IT-004 |
| BR-07 | TC-UT-009 |
| BR-08 | Logical-date evidence: TC-UT-010, TC-IT-003, TC-IT-005, TC-IT-014, TC-IT-015 |
| BR-09 | TC-UT-012 |
| BR-10 | TC-UT-012, TC-IT-006 |
| BR-11 | TC-UT-014 |
| BR-12 | TC-UT-013, TC-IT-006 |
| BR-13 | TC-IT-009 |
| BR-14 | TC-IT-007, TC-DQ-008 |
| BR-15 | TC-IT-009 |
| BR-16 | TC-UT-015 |
| BR-17 | TC-UT-018 |
| BR-18 | TC-UT-018 |
| BR-19 | Payment evidence: TC-UT-016, TC-UT-017, TC-DQ-005, TC-DQ-014 |
| BR-20 | Coverage and dbt evidence: TC-DQ-005, TC-DQ-014, TC-CI-001, TC-CI-002 |
| BR-21 | TC-UT-004 |
| BR-22 | TC-INFRA-003 |
| BR-23 | TC-DQ-011, TC-DQ-012 |
| BR-24 | TC-UI-001 |
| BR-25 | TC-UI-003 |
| BR-26 | TC-INFRA-004 |
| BR-27 | TC-DOC-001 |
| BR-28 | TC-DOC-002 |
| BR-29 | Could scope evidence is trainer-reviewed after all Must gates pass. |
| NFR-PERF-01 | TC-PERF-002 |
| NFR-PERF-02 | TC-PERF-001 |
| NFR-PERF-03 | TC-PERF-003 |
| NFR-SCALE-01 | TC-INFRA-004 and full-data demo step `DEMO-SCALE-01` |
| NFR-REL-01 | TC-UT-011, TC-IT-003 |
| NFR-REL-02 | TC-INFRA-003 |
| NFR-REL-03 | TC-UT-014 |
| NFR-SEC-01 | TC-SEC-001 |
| NFR-SEC-02 | TC-IT-011 |
| NFR-SEC-03 | TC-SEC-002 |
| NFR-PRIV-01 | TC-SEC-003 |
| NFR-MAINT-01 | TC-CI-003 |
| NFR-MAINT-02 | TC-CI-003 |
| NFR-OBS-01 | TC-IT-012 |
| NFR-OBS-02 | TC-IT-013 |
| NFR-USE-01 | TC-UI-001 |
| NFR-ACC-01 | Manual demo step `DEMO-ACC-01` checks readable labels at 1366×768. |
| NFR-PORT-01 | Trainer pre-check step `DEMO-PORT-01` runs lite profile on Windows WSL2, macOS, or Linux. |
| NFR-LIC-01 | TC-DOC-002 |

## Performance-test conditions

| Condition | Value |
|---|---|
| Lite hardware | 4 cores, 8 GB RAM, WSL2 memory `4GB`, swap `4GB`, processors `4` where applicable |
| Standard hardware | 6 or more cores, 16 GB RAM |
| Daily fixture | `fixture_2018_01_02_small`, 63 source rows |
| Full backfill data | About 100,000 Olist orders |
| Warm-up | Start containers and run one small health check before timing |
| Timer start | Airflow DAG run starts |
| Timer stop | Quality summary and final notification complete |
| Network | Full backfill may call Frankfurter outside CI; CI uses fixtures only |
| Pass target | Daily run less than 5 minutes; full backfill less than 30 minutes; CI less than 15 minutes |

## Defect report fields

| Field | Example |
|---|---|
| Defect ID | `BUG-SHOPSIGHT-001` |
| Title | `Duplicate checksum loads accepted rows twice` |
| Environment | `macOS, Docker Compose v5.x, PostgreSQL 18.x` |
| Logical date | `2018-01-02` |
| Batch ID | `20180102T000000Z` |
| Requirement IDs | `FR-RAW-02`, `BR-07` |
| Steps to reproduce | Rerun the same fixture with checksum `abc123`. |
| Expected result | Audit status is `skipped` and accepted count stays `60`. |
| Actual result | Accepted count becomes `120`. |
| Evidence | Airflow task log, audit row, failing test ID |
| Severity | Critical, High, Medium, or Low |
| Owner | Student name |
| Fix verification | Test case ID and CI run link |

## Entry criteria

1. The student repository has the documented folder structure.
2. PostgreSQL 18.x, Airflow 3.3.x, dbt, and test tools start locally.
3. The fixture `fixture_2018_01_02_small` exists with the documented counts.
4. FX fixture files exist for `2018-01-02` and `2018-01-06`.
5. CI can start a PostgreSQL service container.

## Exit criteria

1. All Must test cases in this catalog pass or have trainer-approved evidence.
2. Python line coverage is at least 85%.
3. Python branch coverage is at least 75%.
4. dbt has 0 failing generic, singular, and unit tests.
5. DAG integrity tests pass with 2 retries and 5-minute retry delay.
6. The idempotency test keeps row counts and GMV unchanged for unchanged input checksums.
7. CI finishes in 15 minutes or less on sample data.
8. Documentation tests confirm runbook, data dictionary, and licence attribution.

[Back to README](../README.md)
