# Interview preparation

Purpose: This document helps the student explain ShopSight in interviews and revise the core fundamentals behind the project.

## Explain your project in 2 minutes

Use STAR.

| Part | Script outline |
|---|---|
| Situation | An online marketplace had daily Olist CSV exports and manual spreadsheet reporting. The board needed trusted revenue in INR. |
| Task | I built ShopSight, a local Python batch ELT pipeline with validation, PostgreSQL, dbt, Airflow 3, and JSON/CSV report exports. |
| Action | I generated synthetic Olist-equivalent daily folders, quarantined bad rows, loaded raw tables with committed checksums, used explicit historical FX fixtures or opt-in live rates, built marts, and orchestrated the flow with Airflow. |
| Result | The pipeline safely reruns/corrects logical dates, proves reconciliation, publishes star-schema marts and six-view reports, and runs offline after initial downloads with persisted stop/start state. |

## SQL interview questions

| Question | Key points to cover |
|---|---|
| How would you join orders to customers in Olist? | Join `orders.customer_id` to source customers. Use `customer_unique_id` for customer identity. Avoid counting `customer_id` as a person. |
| What is the difference between `WHERE` and `HAVING`? | `WHERE` filters rows before grouping. `HAVING` filters grouped results. Monthly GMV filters dates before grouping and can filter totals after grouping. |
| How do you calculate monthly GMV? | Use purchase date month. Sum item prices only. Keep freight separate. Convert to INR using the matching FX rate. |
| Why use window functions for seller ranking? | Ranking keeps row detail. `RANK` or `DENSE_RANK` compares sellers per month. It avoids a separate self-join. |
| What is a CTE? | A named temporary query block. It improves readability. Use it for item totals before payment reconciliation. |
| When would you add an index? | Add indexes on join keys, date filters, and idempotency keys. Measure with explain plans. Avoid unnecessary indexes on tiny CI samples. |
| What is a primary key in a mart? | A unique non-null row identifier. `order_item_key` identifies one order item. dbt tests prove uniqueness. |
| How do relationships tests help? | They check fact keys exist in dimensions. They prevent orphan customer, product, seller, and date keys. |
| What is a transaction? | A unit of work that commits or rolls back. Raw replacement for a logical date should not leave partial accepted rows. |
| How do you avoid double counting payments? | Aggregate payments at order grain before joining to item grain. Compare payment total to item plus freight total. |

## Data modelling questions

| Question | Key points to cover |
|---|---|
| What is a star schema? | Central facts and descriptive dimensions. `fct_orders` and `fct_order_items` connect to customer, product, seller, and date dimensions. |
| What is grain? | The meaning of one row. `fct_order_items` is one row per order item. `fct_orders` is one row per order. |
| Why have both `fct_orders` and `fct_order_items`? | Order metrics and payment reconciliation are order grain. Product and seller analytics need item grain. |
| What is SCD Type 1? | Keep only current dimension attributes. Must `dim_customer` stores latest city and state. |
| What is SCD Type 2? | Keep historical versions with valid dates. Should scope tracks customer city or state changes over time. |
| What is a surrogate key? | A warehouse key independent from source keys. It protects marts from messy source identifiers. |
| Why create staging models? | Staging standardizes names and types. It keeps raw data unchanged. It simplifies marts. |
| What is source freshness? | A dbt check that source data arrived recently enough. It catches stale landing or raw data. |

## ETL, ELT, and pipeline questions

| Question | Key points to cover |
|---|---|
| Is ShopSight ETL or ELT? | It validates before load, then transforms in PostgreSQL with dbt. It is best described as batch ELT with ingestion validation. |
| What makes a load idempotent? | Same input and logical date produce the same stored results. Checksums and replacement rules prevent duplicates. |
| How do you handle bad rows? | Write rejected rows to `raw_quarantine`. Keep rule ID, reason, file, source row number, and payload. |
| What is a backfill? | Reprocess historical logical dates. ShopSight backfills a date range and keeps each date idempotent. |
| Why use recorded FX fixtures in tests? | CI must be deterministic and offline. Live APIs can fail or change. |
| What happens after Frankfurter fails? | Airflow retries 2 times with 5-minute delay. After final failure, the task fails and sends an alert. |
| How is local FX different from a fallback? | Fixture mode is selected before the run and reads recorded historical responses. Live mode never changes to fixtures after exhausted retries; missing fixtures also fail clearly. |
| Why use decimal for money? | Floats can introduce precision errors. Decimal values support exact financial totals and rounding. |
| Why round only at mart output? | Early rounding accumulates errors. Mart-level rounding keeps calculations precise. |

## Airflow and dbt questions

| Question | Key points to cover |
|---|---|
| What is a DAG? | A directed acyclic graph of tasks. ShopSight orders wait, load, FX, dbt, quality summary, notify. |
| What is an operator? | A task template that does work. Examples include sensors, Python tasks, and command tasks. |
| What is a sensor? | A task that waits for a condition. ShopSight waits for all 9 landing files. |
| What is XCom? | Small task-to-task metadata. Do not use it for large CSV data. |
| What is a dbt model? | A transformation that creates a table or view. Models are staged into marts with tests. |
| What is a dbt generic test? | Reusable tests such as unique, not_null, relationships, and accepted_values. |
| What is a dbt singular test? | A project-specific SQL assertion. ShopSight uses it for money and date business rules. |
| What is dbt v2? | A newer dbt engine line released after the cohort default. Use it only with an ADR because ecosystem support may differ. |

## Tooling and file-format questions

| Question | Key points to cover |
|---|---|
| CSV versus Parquet: what changes? | CSV is simple and source-like. Parquet is columnar, typed, compressed, and better for analytics. Must scope uses CSV. |
| Why use Docker Compose? | The Python local entry point starts database, Airflow, and Mailpit consistently with loopback ports and persistent project storage. No student frontend is built. |
| Why use Mailpit? | It captures local alert e-mails safely. It avoids sending real e-mail in training. |
| What does SQLFluff do? | It checks SQL style and common anti-patterns. It makes dbt models easier to review. |
| What does pip-audit do? | It checks Python dependencies for known vulnerabilities. CI blocks untriaged serious issues. |
| Why commit `uv.lock`? | It pins dependency resolution. CI and local installs use the same dependency set. |

## Project deep-dive questions

| Question | Key points to cover |
|---|---|
| How did you inject data-quality problems deterministically? | Use seed `20261002`. Keep configured rates stable. Record exact duplicate, null, and bad-date behaviours. |
| How did you prove a row was quarantined for the right reason? | Check `rule_id`, source row number, file name, and payload. Use a test with blank `order_id`. |
| How did you design raw audit? | Store batch ID, logical date, file name, checksum, counts, status, start and finish times. |
| How did you prevent report users from reading raw tables? | The CLI allowlists six analytics views; `report_reader` has SELECT privileges only on those views. Test both forbidden writes and raw SELECTs. |
| How did you test carried-forward FX? | Fixture has a missing weekend day and prior market-day rate. Expected `is_carried_forward=true`. |
| How did you make exports reproducible? | Preserve SQL view columns and filters; stable ordering; decimal strings; exact UTF-8 JSON/CSV snapshots; no volatile timestamps or Python reaggregation. |
| How did you keep the lite profile stable? | One LocalExecutor task and active DAG run, dbt threads 1, no extra UI services. Proposed aggregate budgets include child tasks; measure real usage during trainer pre-check. |
| How did you prove local reliability? | Clean clone after downloads, offline seed/demo, exact `900.00` BRL / `17550.00` INR reports, persisted stop/start checksums/counts, and clear invalid-input/dependency errors. |
| What would break if `customer_id` were used as the customer dimension key? | Repeat customers would be overcounted because Olist uses different `customer_id` values per order. |

## Fundamentals check

| Area | Must know |
|---|---|
| Python core | Functions, exceptions, context managers, dataclasses, type hints, decimals, file paths, package structure. |
| SQL | INNER and LEFT joins, GROUP BY, CTEs, window functions, transactions, indexes, constraints, explain plans. |
| Git | Clone, branch, commit, rebase basics, pull requests, resolving conflicts, tags. |
| Linux | `cd`, `ls`, `grep`, `find`, permissions, environment variables, ports, logs. |
| HTTP and networking | DNS, TCP, HTTP status codes, timeouts, retries, JSON, APIs, local ports. |
| AWS mapping | PostgreSQL maps to RDS. Docker services map to ECS or EC2. Airflow maps to MWAA. Local files map to S3. Logs map to CloudWatch. Secrets map to Secrets Manager. |

## Resume bullets

- Built ShopSight, a batch ELT pipeline that loaded about `<orders_count>` Olist orders into PostgreSQL with idempotent checksum-based ingestion.
- Designed dbt star-schema marts with `<model_count>` models, generic tests on all keys, and `<singular_test_count>` business-rule tests.
- Orchestrated daily and backfill runs in Airflow 3 with 2 retries, 5-minute retry delay, and Mailpit failure alerts.
- Improved data reliability by quarantining `<quarantined_rows>` invalid rows and proving source reconciliation for each load.
- Published six SQL analytics views with tested Python JSON/CSV exports covering GMV, categories, delivery, seller ranking, and payment mix.
- Verified offline local startup and persisted restart behavior through `<local_test_count>` acceptance tests; quote measured RAM only after implementation.

## GitHub and LinkedIn tips

- Pin the `shopsight` repository on GitHub.
- Put the architecture diagram and demo video link near the top of the README.
- Keep issues and PRs public and professional.
- Write a LinkedIn post that explains the business problem, not only the tools.
- Mention exact skills: Airflow 3, dbt, PostgreSQL, Python, SQL windows, Docker, CI.

## Mock-interview checklist

- Explain the project in 2 minutes without reading notes.
- Trace one order from CSV through a SQL view to an exact exported JSON/CSV row.
- Explain a failed dbt test and how you fixed it.
- Write a simple monthly GMV query on a whiteboard.
- Explain idempotency with logical date and checksum.
- Compare ETL and ELT for this project.
- Explain one ADR and one trade-off.
- Show that you can find a failure in Airflow logs.
- Explain doc 06 start/stop commands, loopback bindings, explicit FX modes, and the four local acceptance tests without opening a console.

[Back to README](../README.md)
