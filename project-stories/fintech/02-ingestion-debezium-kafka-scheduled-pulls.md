# Ingestion — Debezium + Kafka (CDC) and Scheduled API/SFTP Pulls

Two ingestion mechanisms feed Bronze, matched to what each source can offer (see `00-problem-statement.md` for what each source holds and `01-pipeline-overview.md` for where this layer sits):

1. **CDC (Change Data Capture)** — for the one source we own and run ourselves (Payments, PostgreSQL). We can read its internal change log directly, so we get every change within seconds.
2. **Scheduled pulls** — for the five vendor/SaaS systems (Mambu, NetSuite, Adyen, Salesforce, Persona). We don't own these systems and have no access to their internal logs, so a REST API (Application Programming Interface) call or an SFTP (Secure File Transfer Protocol) file drop, run on a schedule, is the only option.

```mermaid
flowchart LR
    subgraph CDC[CDC - near real time]
        PG[Payments Postgres - AWS RDS]
        DBZ[Debezium connector reading the WAL]
        KT[Kafka topics, one per table]
    end

    subgraph Scheduled[Scheduled pulls - hourly to daily]
        MAMBU[Mambu API - hourly]
        NETSUITE[NetSuite API + saved-search export - daily]
        ADYEN[Adyen settlement files - SFTP, daily]
        SF[Salesforce Bulk API - daily]
        PERSONA[Persona API + webhooks]
    end

    SINK[Kafka Connect S3 Sink connector]
    LANDING["Landing zone: s3://company-raw/source/date=.../"]
    BRONZE[S3 Bronze - Medallion raw layer]

    PG --> DBZ --> KT --> SINK --> BRONZE
    MAMBU --> LANDING
    NETSUITE --> LANDING
    ADYEN --> LANDING
    SF --> LANDING
    PERSONA --> LANDING
    LANDING --> BRONZE
```

## 1. CDC — Debezium + Kafka (Payments, PostgreSQL)

- **Why this tech:** Payments is our own database, under heavy continuous write load (~5 million transactions a day (confirm)). Polling it on a schedule would mean repeatedly running heavy `SELECT` queries against a live OLTP (Online Transaction Processing) system just to find what changed, which risks slowing down the app customers use to pay. Debezium reads PostgreSQL's WAL (Write-Ahead Log — the database's own internal record of every change, which it already keeps for crash recovery) instead of querying the tables, so it adds almost no load to the primary. It also catches deletes and every intermediate state, which a "give me rows changed since last time" query would miss for rows that were later deleted. See `tech-stack/data-engineering-glossary.md` for the general definitions of CDC, Kafka, and Debezium.
- **What it does in this pipeline:** Debezium's PostgreSQL connector tails the WAL and turns every insert, update, and delete on the `transactions`, `refunds`, and `chargebacks` tables into a JSON/Avro event. Each table's events go to its own Kafka topic (`payments.transactions`, `payments.refunds`, `payments.chargebacks`). A Kafka Connect S3 Sink connector reads those topics and writes the events out as Parquet files directly into S3 Bronze, partitioned by table and date.
  - One row-level change = one event, generated as it happens — nothing is batched here. An `UPDATE` touching 5 columns on one row is still one event; an `UPDATE` touching 200 rows is 200 events, one per row. The only bulk step is the one-time initial snapshot below.
- **Volume/scale:** ~5 million transactions a day, roughly 58 events/second on average, with peaks of 300–500 events/second (confirm) around paydays and promotions. Kafka topics are set to 7 days of retention (confirm) so a downstream consumer can replay recent history if something needs reprocessing. Topics are split into 12 partitions (confirm), keyed by account ID.
- **Key design decisions:**
  - **One topic per source table**, not one shared topic — each table's schema evolves independently, and downstream consumers (for example, the fraud team) can subscribe only to the table they need instead of filtering a firehose. (This is a separate stream per table, not to be confused with the partitioning below, which splits *one* topic into parallel lanes.)
  - **Partition key = account ID, not transaction ID** — every event for one account lands in the same partition, in order. This matters because a customer's balance calculation downstream depends on seeing that account's events in the order they happened; if events were spread across partitions by transaction ID, two consumers could process them out of order.
  - **Confluent Schema Registry with Avro**, set to backward-compatible evolution — Avro messages are just binary bytes with no field names in them, so something has to hold "what does byte range X mean." Instead of repeating that description in every one of the ~5 million daily messages, the schema is stored once in the registry, and each message just carries a small ID pointing to it. This also gives a place to reject an unsafe schema change before it ever reaches Kafka: when the payments team adds a new column, the registry lets it through and old consumers keep working unchanged instead of breaking on an unexpected field.
  - **Initial snapshot before streaming** — when the connector is first set up (or rebuilt), Debezium takes one consistent snapshot of the full existing table (5M+ historical rows) before switching over to live WAL streaming, so Bronze isn't missing all the history that existed before the connector started.
  - **At-least-once delivery, deduplicated downstream** — Debezium/Kafka Connect guarantee at-least-once delivery, not exactly-once, so the same event can land in Bronze twice after a connector restart. Rather than solving this inside the streaming path, we dedupe by `(table, primary key, WAL LSN)` during the Bronze → Silver merge in `03-processing-spark-databricks.md` — simpler than building exactly-once semantics into the ingestion layer itself.
- **Gotchas / what broke (confirm — illustrative until matched to a real incident):**
  - A Schema Registry outage stopped Debezium itself from producing new events to Kafka (it needs the registry to serialize each event, not just the S3 Sink connector reading them back out). With Debezium stuck, it couldn't confirm to Postgres "I'm done with this part of the log," so Postgres had to keep holding onto WAL it would normally discard — and since production kept taking payments the whole time, that unconsumed WAL kept growing and disk usage climbed toward the limit. Two things came out of this:
    - **Alerting:** a monitor checks Postgres's own replication-slot-lag number (how many bytes of WAL are still unconsumed) every minute and pages on-call once it crosses a set size, well before disk actually runs out.
    - **Runbook:** on-call's first move is always to fix the real blocker (get the registry back up) and let Debezium catch up naturally — nothing is lost that way. Only if disk is about to fill before that can happen does the runbook allow the last-resort step: delete the replication slot (this frees the disk immediately, since Postgres can now discard the backlog) and rebuild by creating a new slot plus a fresh Debezium snapshot. The cost: today's current data is fully recovered by the snapshot, but the exact blow-by-blow of what changed during the outage window is gone for good.
  - **Who is Blank?** (a cross-team story)
    - Priya, a backend engineer on the payments team, was tidying up old code and found a column called `merchant_ref`, which records which merchant (shop) each payment went to. It looked unused, so Priya searched the payments code for anything reading it. Zero results. Satisfied, they renamed it to `merchant_reference` and tucked it into a 40-file cleanup pull request. A teammate approved it at 4:55 on a Friday, the official time for approving 40-file pull requests.
    - The search was right: the payments app really didn't use that column. *Our* pipeline did, and our code lives in a different repository, so the search could never find it.
    - Then came the sneaky part: nothing broke.
      - To the Schema Registry, a rename looks like "delete one column, add another," and both are allowed under backward-compatible rules.
      - Back then, our Silver job read a fixed list of columns, so the new `merchant_reference` column was simply ignored, while the old `merchant_ref` arrived empty.
      - Debezium kept streaming, Kafka kept flowing, and every job finished green. But every new payment now had no merchant, and the merchant report put them all into one growing group: "(blank)."
    - A week later, on finance's weekly call, someone read out our top merchant: "Blank." Everyone assumed they meant **Café Blanc**, a tiny French café and one of our merchants, run by its owner, Jean-Luc Blanc. Its account manager started taking credit for the "fastest-growing client in the company." The only problem: Café Blanc had been closed all week, with a sign on the door saying *"Fermé pour vacances,"* while Jean-Luc was on holiday in Lyon. A closed café can't have a record week. Then finance asked the real question: **"Who is Blank?"**
    - The fix was coffee with the payments tech lead, not blame, because nobody had done anything wrong. Their search simply couldn't see our code. Together we agreed on three things, and the data team added a fourth on its own side:
      - **A list of the tables and columns the data platform depends on,** kept in *their* repository, where their search would actually find it.
      - **An automatic reviewer:** any pull request touching those tables (`transactions`, `refunds`, `chargebacks`) adds a data engineer. That's about a day's review, and payments still makes the call.
      - **A polite way to rename:** add the new column, fill both for one sprint while we update our models, then drop the old one.
      - **On our side:** the Silver job now reads every column, so nothing new can be quietly ignored again. (That change exposed the next problem; see the new-column gotcha in `03`.)
    - The next quarter, six schema changes came through, and nothing broke. The payments team even started messaging us *before* opening the pull request. Café Blanc reopened the following Monday, back to its usual 40 croissants a day.

## 2. Scheduled pulls — Mambu, NetSuite, Adyen, Salesforce, Persona

- **Why this tech:** these five systems are vendor-run SaaS (Software as a Service) products — we have an API or a file drop, never database access. A scheduled pull, orchestrated by Airflow (see `04-orchestration-airflow.md`), is the standard way to bring vendor data in at whatever cadence its API limits and our freshness needs allow.
- **What it does in this pipeline:** one Airflow task per source calls the vendor's API (or lists/downloads new files over SFTP), and lands the raw response untouched in an S3 landing prefix, one dated, run-stamped folder per pull, before it's copied into Bronze unchanged.
- **Volume/scale and pull mechanics:**

| Source | Pull mechanism | Frequency | Volume | Pagination / rate limit |
|---|---|---|---|---|
| Mambu | REST API, incremental by `lastModifiedDate` | Hourly | ~500,000 active loans; ~20,000 changed records/hour (confirm) | Cursor pagination, 100 records/page; ~10 requests/sec rate limit (confirm) |
| NetSuite | SuiteQL API + scheduled saved-search export | Daily | ~2,000,000 ledger lines/month, ~65,000/day (confirm) | Day-to-day incremental pulls use the SuiteQL API, which pages normally; the saved-search export path returns only the first 1,000 matching rows per run, with no further page available |
| Adyen | SFTP file drop | Daily, files typically land 2–4 a.m. UTC (confirm) | ~50 CSV files/day | Pipeline lists the SFTP directory and downloads only files not already in its manifest |
| Salesforce | Bulk API (REST) | Daily | ~1,000,000 customer records | Bulk API batches of 10,000 records to stay under Salesforce's governor limits |
| Persona | REST API + webhooks | Webhook on completion; daily API reconciliation pull | ~10,000 checks/day | Webhooks are push-based (no polling); daily pull re-fetches the last 48 hours (confirm) to catch anything a missed webhook dropped |

  *Example:* the Mambu task runs at the top of every hour, asks the API for every loan record with `lastModifiedDate` after the last successful run, pages through the results 100 at a time, and writes the raw JSON to `s3://company-raw/mambu/date=2026-09-28/hour=14/`.

- **Key design decisions:**
  - **Incremental pulls, not full extracts** — Mambu, NetSuite, and Salesforce pulls filter by a last-modified timestamp (or Salesforce's `SystemModstamp`) so each run only pulls what changed. This isn't just an efficiency choice: NetSuite's saved-search export can't finish a full 2-million-row dump inside its execution time limit (confirm), so incremental pulls are the only way it completes reliably.
  - **Idempotent landing** — "idempotent" just means running the same pull twice causes no extra harm. Every pull writes to a uniquely named, date/run-stamped object and never overwrites or appends to a previous one, so a retry after a failure just lands in its own new spot — nothing is lost, nothing is silently doubled up. Deduplication happens later, at the Bronze → Silver merge step, not in ingestion.
  - **SFTP manifest table** — Adyen doesn't guarantee a filename won't be redelivered, so ingestion keeps a manifest (a log) of every filename plus its checksum (a fingerprint of the file's exact content) already processed. Before ingesting a file, the pipeline checks the manifest; a matching filename and checksum means it's already been handled, so it's skipped instead of processed again.
  - **Webhook plus reconciliation for Persona** — a webhook (Persona calling us the moment a check finishes, instead of us repeatedly asking) is efficient, but a failed delivery isn't retried, so relying on it alone risks silently missing a check if our endpoint is briefly unreachable (a deploy, a network blip). The daily reconciliation pull re-fetches recent checks from the API, compares them against what we already have, and fills any gap the webhook stream missed.
  - **Secrets never hard-coded** — every API key and SFTP credential lives in a secrets manager (AWS Secrets Manager (confirm)), referenced by the Airflow connection, not stored in DAG code or config files checked into the repo.
  - **Backoff on rate limits** — every API pull retries on HTTP 429/5xx with exponential backoff, so a throttled Mambu or Salesforce API slows the pull down instead of the task failing outright.
- **Gotchas / what broke (confirm — illustrative until matched to a real incident):**
  - The saved-search export's 1,000-row ceiling was normally invisible, because daily volume stayed under it — until a quarter-end spike in journal entries pushed one day's matching rows past 1,000. The export just silently returned the first 1,000 and dropped the rest, with no error. The pull had no check comparing rows-received against a rough expected count, so the gap wasn't caught until finance's own reconciliation found ledger lines missing three days later. Fix: every pull now logs a rolling average of recent row counts (e.g. last 30 runs) and alerts immediately if a run falls far outside that range, instead of silently accepting whatever came back.
  - Adyen once redelivered the previous day's settlement file — same filename pattern, but a corrected total inside. Before the checksum manifest existed, the pipeline had no way to tell this was a repeat of an already-processed date, so it added the corrected file's total on top of the original instead of replacing it — double-counting that day's settlement amount in Bronze until someone reconciled our totals against the actual bank statement and traced the gap back to it.

## Compliance and security in ingestion

- **PCI-DSS (Payment Card Industry Data Security Standard):** the Payments CDC stream and Adyen's settlement files only ever carry masked or tokenized card data (last 4 digits, processor tokens) — never a raw PAN (Primary Account Number). Ingestion never writes an unmasked card number anywhere, including the landing zone.
- **KYC/AML (Anti-Money Laundering):** Persona's payloads include identity-document data and sanctions-check results. The landing prefix and Bronze location for this source use a narrower IAM (Identity and Access Management) role than the rest of Bronze (confirm exact role name), so only the small set of people who need KYC data for compliance work can read it.
- **SOX (Sarbanes-Oxley Act):** the NetSuite ledger pull is a SOX-relevant control. Every run logs its source, row count, start/end timestamp, and run ID, so an auditor can trace any ledger line all the way back to the exact ingestion run that brought it in.
