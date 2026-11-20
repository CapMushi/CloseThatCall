# Storage and Warehouse — S3 (Bronze, Silver) and Snowflake + dbt (Gold)

This layer is where data lives at each stage. Bronze and Silver are files in the data lake on Amazon S3 (Simple Storage Service). Gold is the certified set of business tables in Snowflake, built with dbt (data build tool).

## Storage: the data lake on S3

- **Why S3:** it's cheap, practically unlimited, and every tool in the pipeline can read it (Kafka Connect, Airflow tasks, Spark, Snowflake). Raw history costs pennies per GB (gigabyte) per month, so we keep everything rather than decide up front what might be needed later.
- **Layout:** one bucket (top-level folder) per layer, organised by source and date so tools only read the dates they need.

| Layer | Bucket (confirm) | Format | Kept for |
|---|---|---|---|
| Bronze | `s3://company-raw/<source>/date=.../` | As received: JSON (JavaScript Object Notation), CSV, Parquet from the CDC (Change Data Capture) sink | 7 years (confirm) |
| Silver | `s3://company-silver/<table>/` | Delta tables (`03-processing-spark-databricks.md`) | Current state, plus 30 days of history |

- **The folders, source by source (confirm).** Bronze answers "what did each source send, and when?", so it's organised by source, then table or feed, then date:
  ```
  s3://company-raw/
    payments/transactions/date=2026-02-14/part-0001.parquet   ← CDC, written continuously
    payments/refunds/date=2026-02-14/...
    payments/chargebacks/date=2026-02-14/...
    mambu/loans/date=2026-02-14/hour=14/run=1400/...           ← hourly pulls
    netsuite/ledger/date=2026-02-14/run=0100/...               ← daily
    adyen/settlements/date=2026-02-15/settlement_2026-02-14.csv
    salesforce/accounts/date=2026-02-14/run=0130/...
    persona/checks/date=2026-02-14/webhook/... and reconciliation/...   ← narrower access role
  ```
  Silver answers "what's the clean, current truth about each business thing?", so it has one Delta table per folder, not split by source or date:
  ```
  s3://company-silver/
    transactions/  refunds/  chargebacks/   (from payments)
    settlements/   (Adyen)    ledger/ (NetSuite)    loans/ (Mambu)
    customers/     (Salesforce + Mambu + payments + Persona, combined)
  ```

- **A "table" on S3 is just files.** S3 only stores files. `silver.transactions` is really a folder of data files plus a small index folder saying which files make up the table right now:
  ```
  s3://company-silver/transactions/
    _delta_log/          ← the index
    part-0001.parquet    ← data files
    part-0002.parquet
  ```
- **Files are never edited; changes add new files.**
  - **New payments:** a new data file is written, and the index adds it, e.g. "version 41: add `part-0007`."
  - **An update,** like T-1001 going from pending to settled: a new file holds the settled row, and the index swaps it in, e.g. "version 42: remove `part-0003`, add `part-0008`."
  - **Reading:** readers follow the latest index entries, so they see only current files. Old files stay for history until a clean-up job deletes them.
- **How Silver is organised:** one bucket with many tables, one folder each, grouped by **business thing**, not by source system.
  - **Combined:** some tables merge several sources. `silver.customers` combines Salesforce, Mambu, and the payments database into one record per customer (`C-000981`). Silver is where the systems first agree.
  - **Mostly one source:** `silver.transactions` (payments database), `silver.refunds`, `silver.settlements` (Adyen), `silver.ledger` (NetSuite), `silver.loans` (Mambu).
- **Bronze has no table format.** It's files exactly as received (JSON, CSV, Parquet), one folder per source and date. Bronze is only ever added to, never updated, so it doesn't need the index; only Silver, where rows change, does.
- **Lifecycle rules to control cost:** Bronze moves to cheaper archive storage (S3 Glacier) after 90 days. It's rarely read after that, but must still exist for audits and replays.

## Getting Silver into Snowflake without copying it

**The decision:** Snowflake reads Silver **in place**; we don't copy it. Snowflake is told "this table lives in these S3 files" and reads those files directly whenever it's queried. It's like sharing one Google Doc link instead of emailing everyone their own copy.

**How:** a **table format** is the index described above: it says which data files make up the table now, plus the history of changes, so plain files behave like a real table. Three names matter:

| | What it is | Example | Its job here |
|---|---|---|---|
| **Delta Lake** | Databricks' open table format; its index is the `_delta_log` folder. It supports updates (`MERGE`), history, and schema checks | Spark merges T-1001's `settled` version into `silver.transactions`, logged as version 42 | Silver is written as Delta, so Spark can update and de-duplicate safely (`03`) |
| **Apache Iceberg** | Another open table format, read by many engines (Snowflake, Spark, Trino, Athena). Its index also stores statistics, such as the date range in each file | Snowflake queries February, sees from the index which files hold only January, and skips them | The "language" Snowflake uses to read Silver in place |
| **UniForm** (Universal Format) | A Delta setting that, after each Delta change, also writes Iceberg index files for the same data files | Spark commits version 42; UniForm writes matching Iceberg metadata, so Snowflake's next read sees `settled` (after a short delay, confirm) | The bridge: one copy of the data, readable by both Spark and Snowflake |

In Snowflake, the Silver tables are registered as Iceberg tables in a `silver` schema (confirm).

**Why:** copying would mean two Silvers that could fall out of step, a sync job, and double the storage bill. Reading in place gives one Silver, with no sync.

**Trade-off:** Snowflake reads its own native tables faster than outside files. So dbt reads Silver once, while building Gold, and stores **Gold as native Snowflake tables**. Dashboards only ever hit the fast native tables.

**Where we *do* copy into Snowflake:**
- **Building Gold** writes transformed data into Snowflake's own storage, for speed.
- **If Silver couldn't be read in place,** the standard tool would be **Snowpipe**, Snowflake's auto-loader, which copies each new S3 file into a table as it lands.

## Warehouse: Gold in Snowflake

- **What Snowflake is:** a cloud data warehouse, a database service built for analytics that we rent. It stores the tables, provides the machines that run queries, runs the SQL (Structured Query Language), and handles security. You *can* type SQL into its web screen, but our build SQL lives in Git and dbt sends it.
- **Who queries it:**
  - **Tableau** fetches its numbers from Snowflake by running SQL; every refresh is a query.
  - **dbt's** nightly build is the heaviest work.
  - **Finance analysts** run checks during month-end close.
  - **Risk** asks ad-hoc exposure questions ("loans 30+ days late, by segment").
  - **Data science** pulls training data.
- **Why Snowflake:** all of those hit it at once. Snowflake separates storage from compute: each workload gets its own warehouse (see Cost below), so the nightly build never slows down an analyst's big query. It also has the governance features this story needs: masking, Time Travel, cloning (`08-compliance.md`).
- **What dbt is:** a tool, not a language; the language is SQL.
  - Each Gold table is a `SELECT` query in its own file, e.g. `mart_revenue_monthly.sql`.
  - dbt works out the build order from which tables each query uses, sends each query to Snowflake (which creates the table), then runs the tests listed next to it.
- **Why dbt, and how Git fits:** "how is revenue calculated?" is answered by pointing to one reviewed file, not someone's memory. A change goes like this:
  1. An engineer edits the file and opens a pull request (a request for a teammate to review).
  2. An automatic check runs dbt and its tests on a test copy of the data.
  3. The reviewer approves.
  4. That night, Airflow runs dbt with the merged files.
  Git records who changed what, and why.

### Schema and modelling approach

**Star schema:** we use a dimensional model, the standard layout for reporting.
- **Fact tables** hold the events we measure, one row per event.
- **Dimension tables** hold the context around them.
- Each fact table has a stated **grain**: what exactly one row means. It tells you how the numbers may be added up.
  - In `fct_loan_balance_daily`, one row is one loan on one day.
  - **Summing across loans** for one day is right: on 28 Feb, L-778 owes \$50,000 and L-905 owes \$18,300, so \$68,300 is owed in total.
  - **Summing across days** is wrong: each loan appears once per day, so "total owed in February" would come out about 28 times too big. A balance is a snapshot, so across time you take one day (e.g. month-end) or an average.
  - Payments are different: each row is a separate event, so summing across days is fine.
- **Name prefixes:**
  - `fct_` is a fact, a list of events.
  - `dim_` is a dimension, a descriptive "card" about a customer or loan.
  - `mart_` is a **data mart**: a ready-made, already-summarised table for one business use. Facts and dimensions are the ingredients; the mart is the prepared meal that dashboards read directly, instead of adding up billions of payment rows.

```mermaid
flowchart LR
    CUST[dim_customer: one row per customer]
    LOAN[dim_loan: one row per loan, its terms]
    BAL[fct_loan_balance_daily: one row per loan per day]
    PAY[fct_payments: one row per payment]
    LED[fct_ledger_lines: one row per NetSuite ledger line]
    REV[mart_revenue_monthly: certified revenue, reconciled]

    CUST --> LOAN --> BAL
    CUST --> PAY --> REV
    LED --> REV
```

Every fact table also links to `dim_date` (one row per day, with month, quarter, fiscal period, and the Eastern business date finance uses for "a day"). It's left out of the picture to keep it readable.

| Gold table | Grain | Built from (Silver) | What it is, with Jane's example, and how it differs |
|---|---|---|---|
| `fct_payments` | One payment | `silver.transactions` + `silver.refunds` + `silver.settlements` | Every card payment we process for a merchant. T-1001: Jane's customer paid \$100. We charged Jane \$3.00, Adyen charged us \$2.10, so we earned **\$0.90**. Jane's later \$5 refund only changes what she keeps (\$92); we keep our fee (confirm).<br><br>**Differs from** `payments_intraday`: certified and nightly, with refunds and fees. From `fct_ledger_lines`: our record of each payment, while the ledger is accounting's separate record. |
| `fct_loan_balance_daily` | One loan, one day | `silver.loans` | A daily photo of every loan: how much is owed, how late. L-778 on 28 Feb: \$50,000 owed, 0 days late (signed 16 Feb, first repayment in March).<br><br>**Differs from** `fct_loan_income`: a state, so never summed across days. From `dim_loan`: what's owed today, not what was agreed. |
| `fct_loan_income` | One interest or fee charge on a loan, or its refund | `silver.loans` (Mambu) | Lending revenue: interest on each repayment date (the main part), origination and late fees, and refunds. Jane paid a \$1,000 origination fee on L-778.<br><br>**Differs from** `fct_loan_balance_daily`: money we earn, summed into revenue, not money still owed. |
| `fct_ledger_lines` | One ledger line | `silver.ledger` (NetSuite) | Accounting's official book: one amount posted to one account (e.g. 4000 Card fee revenue), as **daily totals per product**. Jane's \$3.00 fee is inside the 15 Feb card-fee line of \$4.48M.<br><br>**Differs from** `fct_payments`: the same money recorded by accounting, so the two are compared by month and product, not payment by payment. |
| `dim_customer` | One customer | `silver.customers` | One card per customer (a new row only when a tracked detail changes, see type 2 below). C-000981: Jane's Bakery LLC, small business, joined March 2025.<br><br>**Differs from** `dim_loan`: who the customer is; one customer can have several loans. |
| `dim_loan` | One loan | `silver.loans` | One card per loan, with terms fixed at signing. L-778: \$50,000, 12.5%, 48 months. These numbers describe the loan (to filter and group, e.g. "loans over \$25,000"), so they belong in a dimension.<br><br>**Differs from** `fct_loan_balance_daily`: what was agreed, not what's owed today. |
| `mart_revenue_monthly` | One month, one product | `fct_payments` + `fct_loan_income` + `fct_ledger_lines` | **The certified revenue number**, checked against accounting. February card payments include Jane's \$0.90 and match NetSuite.<br><br>**Differs from** `mart_kpis_monthly`: money, checked against NetSuite; the KPI mart holds counts and rates, with no accounting version to check. |
| `mart_kpis_monthly` | One month | `fct_payments` + `fct_loan_balance_daily` | The other two headline numbers: **active customers** (a settled payment in the last 90 days) and **default rate** (share of loans 90+ days late on the month's last day) (confirm both). Jane is active (paid 15 Feb); L-778 is healthy (0 days late).<br><br>**Differs from** `mart_revenue_monthly`: see that row. |
| `payments_intraday` | One payment, today only | `silver.transactions` | **Provisional** gross volume and payment counts, refreshed every 15 minutes (see below). T-1001 appears at 13:52 on 14 Feb as \$100, pending.<br><br>**Differs from** `fct_payments`: no refunds, fees, or net, and never certified; the next night the same payment is in `fct_payments`. |


```mermaid
%%{init: {'themeCSS': 'td, th { border: 1px solid #999; padding: 2px 6px; white-space: nowrap; font-size: 12px; } table { border-collapse: collapse; margin-top: 4px; }'}}%%
flowchart TB
    CUST["<b>dim_customer</b><table><tr><th>customer_id</th><th>business_name</th><th>segment</th><th>valid_from</th><th>valid_to</th></tr><tr><td>C-000981</td><td>Jane's Bakery LLC</td><td>small business</td><td>2025-03-02</td><td>2026-06-01</td></tr><tr><td>C-000981</td><td>Jane's Bakery LLC</td><td>mid-market</td><td>2026-06-01</td><td>(current)</td></tr><tr><td>C-001544</td><td>Tom's Tacos LLC</td><td>small business</td><td>2024-08-10</td><td>(current)</td></tr></table>"]
    INTRA["<b>payments_intraday</b> (today only, provisional)<table><tr><th>txn_id</th><th>customer_id</th><th>created_at</th><th>gross</th><th>status</th></tr><tr><td>T-1001</td><td>C-000981</td><td>2026-02-14 13:52</td><td>100.00</td><td>pending</td></tr><tr><td>T-1002</td><td>C-001207</td><td>2026-02-14 13:58</td><td>40.00</td><td>pending</td></tr></table>"]
    LOAN["<b>dim_loan</b><table><tr><th>loan_id</th><th>customer_id</th><th>principal</th><th>rate_pct</th><th>term_months</th><th>status</th></tr><tr><td>L-778</td><td>C-000981</td><td>50,000.00</td><td>12.5</td><td>48</td><td>active</td></tr><tr><td>L-905</td><td>C-001544</td><td>20,000.00</td><td>14.0</td><td>24</td><td>active</td></tr></table>"]
    INC["<b>fct_loan_income</b><table><tr><th>income_id</th><th>loan_id</th><th>income_type</th><th>income_date</th><th>amount</th></tr><tr><td>F-3301</td><td>L-778</td><td>origination_fee</td><td>2026-02-16</td><td>1,000.00</td></tr><tr><td>I-5102</td><td>L-905</td><td>interest</td><td>2026-02-16</td><td>213.50</td></tr><tr><td>F-4410</td><td>L-905</td><td>late_fee</td><td>2026-02-21</td><td>25.00</td></tr><tr><td>F-4498</td><td>L-612</td><td>fee_refund</td><td>2026-02-28</td><td>−475.00</td></tr></table>"]
    BAL["<b>fct_loan_balance_daily</b><table><tr><th>loan_id</th><th>balance_date</th><th>outstanding</th><th>days_past_due</th></tr><tr><td>L-778</td><td>2026-02-16</td><td>50,000.00</td><td>0</td></tr><tr><td>L-778</td><td>2026-02-28</td><td>50,000.00</td><td>0</td></tr><tr><td>L-905</td><td>2026-02-16</td><td>18,300.00</td><td>0</td></tr><tr><td>L-905</td><td>2026-02-28</td><td>18,300.00</td><td>12</td></tr></table>"]
    PAY["<b>fct_payments</b><table><tr><th>txn_id</th><th>customer_id</th><th>settled_date</th><th>gross</th><th>refund</th><th>our_fee</th><th>processor_fee</th><th>net_revenue</th></tr><tr><td>T-1001</td><td>C-000981</td><td>2026-02-15</td><td>100.00</td><td>5.00</td><td>3.00</td><td>2.10</td><td>0.90</td></tr><tr><td>T-1002</td><td>C-001207</td><td>2026-02-15</td><td>40.00</td><td>0.00</td><td>1.20</td><td>0.84</td><td>0.36</td></tr></table>"]
    LED["<b>fct_ledger_lines</b> (NetSuite, daily totals)<table><tr><th>ledger_line_id</th><th>posting_date</th><th>account</th><th>product</th><th>amount</th></tr><tr><td>NS-55012</td><td>2026-02-15</td><td>4000 Card fee revenue</td><td>card_payments</td><td>4,480,000.00</td></tr><tr><td>NS-55013</td><td>2026-02-15</td><td>5100 Processor fees</td><td>card_payments</td><td>−2,985,700.00</td></tr><tr><td>NS-56200</td><td>2026-02-16</td><td>4200 Loan interest</td><td>lending</td><td>1,148,600.00</td></tr><tr><td>NS-58120</td><td>2026-03-01</td><td>4110 Loan fee refunds</td><td>lending</td><td>−475.00</td></tr></table>"]
    KPI["<b>mart_kpis_monthly</b><table><tr><th>month</th><th>active_customers</th><th>open_loans</th><th>loans_90_plus</th><th>default_rate</th></tr><tr><td>2026-01</td><td>405,249</td><td>498,200</td><td>11,460</td><td>2.3%</td></tr><tr><td>2026-02</td><td>410,112</td><td>500,000</td><td>11,500</td><td>2.3%</td></tr></table>"]
    REV["<b>mart_revenue_monthly</b><table><tr><th>month</th><th>product</th><th>net (platform)</th><th>net (ledger)</th><th>difference</th><th>status</th></tr><tr><td>2026-02</td><td>card_payments</td><td>41,842,310.00</td><td>41,842,310.00</td><td>0.00</td><td>reconciled</td></tr><tr><td>2026-02</td><td>lending</td><td>32,211,940.00</td><td>32,212,415.00</td><td>−475.00</td><td>explained: timing</td></tr></table>"]

    CUST -->|customer_id| LOAN
    LOAN -->|loan_id| INC
    LOAN -->|loan_id, every day| BAL
    CUST -->|customer_id| PAY
    INTRA -->|overnight, once settled: certified| PAY
    BAL -->|month-end day: default rate| KPI
    PAY -->|active customers| KPI
    KPI ~~~ REV
    INC -->|amount summed| REV
    PAY -->|net_revenue summed| REV
    LED -->|summed, then compared| REV
```

- **The certified revenue model is the heart of problem #1.**
  - `mart_revenue_monthly` computes net revenue once. For card payments it's our fee − Adyen's fee: Jane's \$100 payment earns us \$0.90. The \$100 is gross payment volume, mostly Jane's money, and her \$92 after the refund and our fee is what she keeps; neither is our revenue. For lending it's interest + fees − refunds.
  - It then compares that month's total with NetSuite's ledger. Any gap must be explained by a known timing difference (e.g. a refund booked in the next month). Anything larger fails a dbt test.
  - Finance signed off this model, and that sign-off is what "certified" means.
  - **Why not just take revenue from NetSuite?**
    - The ledger holds daily totals with no customer or segment detail, while product and risk need to drill down to payments.
    - It arrives once a day and is adjusted during close.
    - Two independent sources agreeing is the check itself: when they differ, something is broken (a missing refund, a payment never booked), and we find it at close instead of an auditor finding it.
- **Fresh data during the day (provisional, not certified).**
  - `01` promises transaction data to analysts within minutes, but certified net revenue can't be final the same day: it needs Adyen's fees and NetSuite's ledger, which arrive once a day.
  - So a small dbt model, `payments_intraday`, rebuilds every 15 minutes from `silver.transactions`, right after each Silver run (`04`). It holds today's gross volume and payment counts only, and is clearly labelled provisional.
  - Certified revenue stays nightly.
- **dbt layers:**
  - **staging:** light renaming over the Silver tables;
  - **intermediate:** joins, e.g. payment + refund + settlement;
  - **marts:** the Gold tables above.
- **History where it matters (slowly changing dimension, type 2).**
  - **The problem:** Jane's Bakery is "small business" in March and grows to "mid-market" in June. If we simply overwrite the segment, March's revenue would move under mid-market, and a report finance already signed off would change after the fact.
  - **The fix:** a **dbt snapshot** compares customer data on every run and records each change with dates, so March's payments join to the version valid in March:

    | customer | segment | valid from | valid to |
    |---|---|---|---|
    | C-000981 | small business | 2025-03-02 | 2026-06-01 |
    | C-000981 | mid-market | 2026-06-01 | (current) |

  - **Type 1** also exists: it overwrites and keeps no history. These are just numbered methods, not "errors." To choose, ask "was the old value ever true?"
    - **No, it was a mistake:** a business name typed as "Janes Bakry LLC," or a product mapped to the wrong category. That's type 1: overwrite, so every report, past ones included, shows the corrected value.
    - **Yes, it was true for a while:** the segment. That's type 2: keep history.
- **Tests on every build:**
  - `txn_id` (transaction ID, e.g. `T-1001`) is unique and never empty, so no payment is counted twice.
  - Every payment's `customer_id` exists in `dim_customer`.
  - `status` is only `pending`, `settled`, `failed`, or `disputed`. If a word like "in process" slips through a missed mapping, the test fails, instead of that payment quietly vanishing from every "settled" total.
  - **Revenue reconciliation:** each month's Gold net revenue matches the NetSuite ledger. Small gaps with a known cause, like a refund booked the next month, are allowed; anything bigger fails.
  - All are `severity: error`. If one fails, the build fails and Airflow stops before refreshing Tableau, so dashboards keep yesterday's correct numbers. The failure is posted to Slack `#data-alerts` and pages on-call (`04-orchestration-airflow.md`). `severity: warn` results only get a Slack summary.

### Cost and performance tuning

**The terms:**
- **Warehouse:** in Snowflake a warehouse is *not* storage. It's a rented group of computers that runs queries.
- **Size:** extra-small, small, medium, and so on. Each step up doubles the machines and the cost: 1, 2, 4 credits an hour.
- **Credit:** Snowflake's billing unit (price per credit depends on the contract, confirm).
- **Auto-suspend:** a warehouse switches itself off after 60 seconds with no queries, and back on in about a second when one arrives.

| Warehouse (confirm) | Used by | Size | Notes |
|---|---|---|---|
| `TRANSFORM_WH` | dbt nightly build | Medium | Runs about 40 minutes a night (≈ 2.7 credits), then pauses |
| `BI_WH` | Tableau | Small | Extract refreshes after the nightly build, plus the live today's-payments dashboard |
| `ADHOC_WH` | Analysts | Extra-small | Pauses after 60 seconds idle |
| `INTRADAY_WH` | The 15-minute `payments_intraday` build | Extra-small | About a minute per run, then pauses |

- **`TRANSFORM_WH` vs `BI_WH`:** think of a kitchen and a dining room.
  - `TRANSFORM_WH` is the kitchen: it *builds* (writes) Gold, with a few heavy jobs overnight.
  - `BI_WH` is the dining room: it *serves* (reads) Tableau: the extract refreshes and the live today's-payments dashboard.
  - Keeping them apart means a slow build never slows Tableau, and we can see what each costs.
- **Pausing:** every warehouse pauses itself after 60 seconds idle, so we don't pay for machines doing nothing.
- **Resource monitors** are budget alarms on a warehouse. For example, `ADHOC_WH` gets 300 credits a month (confirm): at 80% (240) we get an email, and at 100% Snowflake suspends it. Only ad-hoc can be switched off automatically. Suspending the nightly build or Tableau would break finance's dashboards, so those only send the email.
- **Incremental models.** dbt can build a table two ways:
  - **Full rebuild:** recompute it from all of Silver every night. That's fine for small tables like `dim_loan` (500,000 rows).
  - **Incremental:** keep the existing table and only process what's **new or changed**. That's essential for `fct_payments` (about 1.8 billion rows a year, confirm), where a nightly rebuild would be slow and expensive.
  - **How "changed" is found:** each Silver row has a `last_changed` time. Each night dbt takes payments that changed since the last build: today's new payments, *plus* older payments that got a refund or settlement today. It inserts the new ones and updates the changed ones (`MERGE`). *Example:* T-1001 was paid on 14 Feb, got Adyen's \$2.10 fee from the 15 Feb settlement file, and was refunded \$5 on 20 Feb, so it counts as changed on each of those builds and its row is updated each time.
  - **Lookback window:** a safety margin. dbt reprocesses everything changed in the last **3 days** (confirm), not just since exactly the last build, to catch rows that arrived late or slipped past a failed run. Re-merging them is harmless, because `MERGE` never duplicates.
- **Clustering:** Snowflake stores a table in many small chunks. If February's payments were scattered across every chunk, a February query would open them all. Clustering `fct_payments` by settlement date keeps each day's payments together, so a February query opens only February's chunks. It's like filing receipts in folders by date instead of in one box; the same idea as the date folders in Bronze.
- **Tableau extracts:** Tableau can either ask Snowflake *live* on every filter click (a query, and a cost, each time) or keep its own snapshot, an *extract*. An extract works like Tableau's own cache, refreshed on our schedule. We refresh extracts once after each nightly build, so clicks read Tableau's copy: faster, and free on the Snowflake side. Gold changes only once a day, so the extract is just as fresh.

## Gotchas (confirm — illustrative until matched to a real incident)

- **Late Adyen fees didn't reach Gold.**
  - **What happened:** the first version of the incremental `fct_payments` model chose rows by *payment date*: "payments made since the last build." Adyen's \$2.10 fee for T-1001 arrived in the 15 Feb settlement file, but the payment was made on 14 Feb, so it was never picked up, and the row kept \$3.00 of net revenue instead of \$0.90. Every payment's Adyen fee arrives a day later like this, so February's net revenue came out far too high, and the reconciliation test caught the gap with NetSuite at close.
  - **Fix:** the model now chooses rows by *last changed* time (of the payment, its refund, or its settlement) with a 3-day safety window, as described under Incremental models. A settlement or refund days later still updates its payment, because that change itself happens that day.

Compliance and security for this layer are in `08-compliance.md`.
