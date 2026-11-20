# Ingestion

## What it does

Ingestion gets data out of the six source systems (`01-overview.md`) and into Bronze, the raw layer on S3 (Simple Storage Service), without changing it. It runs on two paths:

| | Live path | Scheduled path |
|---|---|---|
| **Source** | Our own payments database (PostgreSQL) | The five vendor systems: Mambu, NetSuite, Adyen, Salesforce, Persona |
| **How** | CDC (Change Data Capture): Debezium reads the database's change log and streams it through Kafka | Airflow tasks call each vendor's API (Application Programming Interface) or download its files |
| **Speed** | Kafka in seconds, Bronze in a few minutes | Hourly or daily |

Why two paths? We own the payments database, so we can read its change log directly. The vendors only give us an API or a file drop, never their database, so the best we can do is ask on a schedule.

## Why these tools (and what we didn't pick)

**Live path**
- **Debezium.** PostgreSQL already writes every change to its WAL (Write-Ahead Log), which it keeps for crash recovery. Debezium reads that log instead of querying the tables, so it adds almost no load to the database customers are paying through. It also sees every change, including deletes.
  - *Not scheduled `SELECT` queries:* asking "what changed since last time?" over and over puts heavy queries on a live database, and it misses rows that were deleted or changed twice between two runs.
- **Kafka.** It keeps every change for 7 days, so anything downstream can replay after a failure. More than one team can read the same stream: our pipeline writes it to Bronze, and the risk team watches it live. We run it on Amazon MSK (Managed Streaming for Apache Kafka), so we don't manage the servers, and Debezium runs next to it on MSK Connect.

**Scheduled path**
- **Airflow tasks.** Airflow already orchestrates the rest of the pipeline (`04-orchestration-airflow.md`), so each pull is one more task with the same retries and alerts.
  - *Not a managed connector service such as Fivetran:* those charge by the number of rows moved, and Adyen's reports alone add millions of rows a day.

## How it works

### Live path: one payment

1. **The app writes the payment.** A customer pays Jane's Bakery \$100. The payments app inserts a row into the `transactions` table.
2. **Debezium turns the change into an event.** It reads the insert from the WAL and publishes one event to the Kafka topic `payments.public.transactions`. Every insert, update and delete on a row becomes its own event. The `refunds` and `chargebacks` tables each have their own topic, so a team can read only the table it needs.
3. **A second connector writes it to Bronze.** The Kafka Connect S3 Sink reads the topics and writes Parquet files to S3 every few minutes.

**Inside Kafka**

| Topic | Fed by table | Read by |
|---|---|---|
| `payments.public.transactions` | `transactions` | S3 Sink, risk team |
| `payments.public.refunds` | `refunds` | S3 Sink |
| `payments.public.chargebacks` | `chargebacks` | S3 Sink, risk team |

- **A message has a key and a value.** The key is the row's primary key (`T-1001`). The value is the change event shown under Sample data.
- **Each topic is split into 12 partitions,** so several readers can work in parallel. Kafka only keeps messages in order *inside* one partition, and it picks the partition from the key. So every event for `T-1001` lands in the same partition, and "pending" always arrives before "settled".
- **Why the risk team reads Kafka, not the payments database.** Watching the database live would mean running queries on it every few seconds, on top of the payments it's already handling. Kafka hands them every change as it happens, and the database never feels it.
- **Each reader keeps its own place.** The S3 Sink and the risk team read as two separate consumer groups. Kafka stores each group's position (its offset) per partition, so one never slows the other, and after a restart the sink carries on from where it stopped.
- **Messages stay for 7 days,** read or not, and every partition is copied to 3 servers, so losing one server loses nothing.

Three more choices that matter:
- **A snapshot first.** When Debezium is first set up, it copies the existing tables once, then switches to reading the log. Bronze starts with the full history, not just changes from that day on.
- **A schema registry.** Events are stored as Avro, a compact binary format. The AWS (Amazon Web Services) Glue Schema Registry holds the list of columns once, instead of repeating it in every message, and it rejects a change that would break existing readers.
- **Duplicates are fine here.** After a restart, Debezium can send an event twice. We don't fight that in ingestion. Every event carries its LSN (Log Sequence Number), its position in the WAL, and the Silver job uses it to drop repeats (`03-processing-spark-databricks.md`).

### Scheduled path: five vendors

| Source | How we pull | How often | What we ask for |
|---|---|---|---|
| Mambu | REST API | Hourly | Loans changed since the last successful run (`lastModifiedDate`) |
| NetSuite | SuiteQL (NetSuite's SQL-over-API) | Daily | Ledger lines changed since the last run, one page of up to 1,000 rows at a time |
| Adyen | SFTP (Secure File Transfer Protocol) | Daily | New settlement reports. A sensor waits for them to appear. |
| Salesforce | Bulk API | Daily | Customer records changed since the last run (`SystemModstamp`) |
| Persona | Webhooks, plus a daily REST API pull | As each check finishes, then daily | Every finished check, then all checks from the last 2 days again |

Choices that matter:
- **Only what changed.** Mambu, NetSuite and Salesforce are filtered by a last-modified time, so each run is small.
- **Every pull lands in its own folder.** A run never overwrites an earlier one, so a retry can't lose or damage anything. Bronze may then hold the same record twice, and Silver removes the duplicates (`03`).
- **Each Adyen file is loaded once.** A manifest table lists every file already downloaded, with its checksum (a fingerprint of the content). A file that's already in the manifest is skipped.
- **Persona pushes, and we double-check.** Airflow can't receive webhooks, so a small always-on endpoint (Amazon API Gateway in front of a Lambda function) accepts each one and writes it to Bronze. Persona retries a failed delivery for about a day and a half, then gives up. The daily pull fetches the last 2 days again and fills any gap.
- **Slow down, don't fail.** When a vendor says "too many requests", the task waits and retries with growing gaps.
- **Count the rows.** Every pull compares its row count with the average of its last 30 runs and fails if it's far off. The reason is the NetSuite story below.

### Where it lands

```
s3://company-raw/
  payments/transactions/date=2026-02-14/part-0001.parquet   ← live path, written every few minutes
  payments/refunds/date=2026-02-14/...
  payments/chargebacks/date=2026-02-14/...
  mambu/loans/date=2026-02-14/hour=14/run=1400/...           ← one folder per pull
  netsuite/ledger/date=2026-02-14/run=0100/...
  adyen/settlements/date=2026-02-15/settlement_2026-02-14.csv
  salesforce/accounts/date=2026-02-14/run=0130/...
  persona/checks/date=2026-02-14/webhook/...  and  reconciliation/...
```

Nothing in Bronze is ever edited. How the data is cleaned is in `03`; how Bronze is stored and protected is in `05-storage-warehouse-s3-snowflake.md` and `08-compliance.md`.

## What broke

**The change log that nearly filled the disk.** PostgreSQL keeps its WAL until Debezium confirms it has read it. One weekend the Debezium connector failed and nobody noticed. The payments app kept taking payments, the unread log kept growing, and the database's disk climbed towards full, which would have stopped payments.
- *Fix:* an alert on the size of the unread log pages on-call long before the disk is at risk, and another fires when the connector stops (`07-monitoring-observability-datadog.md`). On-call restarts the connector, and it catches up from where it stopped, so nothing is lost.

**The pull that "succeeded" with 1,000 rows.** SuiteQL returns at most 1,000 rows per page and says whether more are waiting. A change to our paging code made the NetSuite pull stop after the first page. There was no error and the task was green. Finance found the missing ledger lines three days later, during their own checks.
- *Fix:* the row-count check above. A pull that returns far fewer rows than usual now fails loudly.

**Who is Blank?** (a cross-team story)
- Priya, a backend engineer on the payments team, was tidying up old code and found a column called `merchant_ref`, which records which merchant each payment went to. It looked unused, so Priya searched the payments code for anything reading it. Zero results. They renamed it to `merchant_reference` inside a 40-file cleanup pull request. A teammate approved it at 4:55 on a Friday, the official time for approving 40-file pull requests.
- The search was right: the payments app didn't use that column. *Our* pipeline did, and our code lives in a different repository, so the search could never find it.
- Then came the sneaky part: nothing broke. To the schema registry, a rename looks like "remove one column, add another", and both are allowed. Our Silver job read a fixed list of columns, so it ignored the new one while the old one arrived empty. Every job finished green, and every new payment went into one growing group on the merchant report: "(blank)".
- A week later, on finance's weekly call, someone read out our top merchant: "Blank." Everyone assumed they meant **Café Blanc**, a tiny French café run by its owner, Jean-Luc Blanc. Its account manager started taking credit for the fastest-growing client in the company. The only problem: Café Blanc had been closed all week, with a sign on the door saying *"Fermé pour vacances"*. A closed café can't have a record week. Then finance asked the real question: **"Who is Blank?"**
- The fix was coffee with the payments tech lead, not blame, because nobody had done anything wrong. We agreed on three things, and added a fourth on our side:
  - **A list of the tables and columns the data platform depends on,** kept in *their* repository, where their search finds it.
  - **An automatic reviewer:** any pull request touching `transactions`, `refunds` or `chargebacks` adds a data engineer.
  - **A polite way to rename:** add the new column, fill both for one sprint while we update our side, then drop the old one.
  - **On our side:** the Silver job now reads every column. (That turned out not to be enough; see `03`.)
- The next quarter, six schema changes came through and nothing broke. The payments team started messaging us *before* opening the pull request. Café Blanc reopened the following Monday, back to its usual 40 croissants a day.

## Sample data

**The event Debezium publishes for Jane's payment** (trimmed to the main fields):

```json
{"op": "c",
 "before": null,
 "after": {"txn_id": "T-1001", "account_id": "acct_77812", "amount": 100.00, "status": "pending"},
 "source": {"table": "transactions", "lsn": 500}}
```

- `op`: what happened. `c` is a new row, `u` an update, `d` a delete.
- `before` and `after`: the row before and after the change. A new row has no "before".
- `lsn`: the change's position in the WAL. When the payment later settles, a second event arrives with `"op": "u"`, `"status": "settled"` and a higher `lsn`.

**One row of a Mambu pull**, Jane's new loan (`mambu/loans/date=2026-02-16/hour=14/run=1400/`):

```json
{"id": "L-778", "accountHolderKey": "12345", "loanAmount": 50000.00,
 "accountState": "ACTIVE", "lastModifiedDate": "2026-02-16T13:41:00+00:00"}
```

`accountHolderKey` is Mambu's own ID for Jane. It's different from her payments account (`acct_77812`) above, which is exactly the mismatch Silver has to resolve.

Compliance and security for this layer are in `08-compliance.md`.
