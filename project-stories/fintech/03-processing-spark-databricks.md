# Processing

## What it does

Processing turns Bronze (raw, exactly as each source sent it) into Silver: one clean table per business thing, such as `silver.transactions`, `silver.loans` and `silver.customers`. Silver is where the six systems first agree. Four things happen on the way:

| Clean-up | Before (Bronze) | After (Silver) |
|---|---|---|
| **One customer ID** | Salesforce `CUST-4821`, Mambu `12345`, payments `acct_77812` | `C-000981` everywhere |
| **One clock** | Payments in UTC (Coordinated Universal Time), Adyen's file in `CET` (Central European Time) | Everything in UTC |
| **One set of status words** | Payments `disputed`, Adyen `Chargeback` | `pending`, `settled`, `failed` or `disputed` |
| **No duplicates** | The same record can arrive twice | One row per payment, loan or customer |

## Why these tools (and what we didn't pick)

- **Apache Spark** spreads the work across a group of machines instead of one. Most days one machine could cope, but re-processing a month of payments (about 150 million rows) after a logic fix could not.
- **Databricks** runs Spark for us: it starts the machines, runs the job and shuts them down. The machines run inside our own AWS (Amazon Web Services) account, so the data never leaves it.
- **Delta Lake** is the table format Silver is stored in. Plain files can only be added to. Delta lets a job *update* a row with `MERGE` (update the row if it exists, insert it if it doesn't). We need that because the same payment changes over time: `T-1001` arrives as `pending`, later as `settled`, and Silver must hold one row, not two. How Delta works underneath is in `05-storage-warehouse-s3-snowflake.md`.
- *Not Snowflake for this step:* either platform could do the whole pipeline, so we split the work by strength. Bronze is raw files in three formats, and cleaning it needs real code, such as the name matching below, which is written in Python. Spark is built for both. Snowflake is built to answer SQL from many people at once, so it gets Gold. Silver sits between them as open files on our S3 (Simple Storage Service) that both can read.
- *Not an always-on streaming job for payments:* a run every 15 minutes is fresh enough for the dashboards, and it costs far less than machines running all day. The risk team, who need seconds, read Kafka directly (`02-ingestion-debezium-kafka-scheduled-pulls.md`).

## How it works

### One run of the payments job

1. **Read only the new files.** Databricks Auto Loader remembers which Bronze files it has already read, in a small checkpoint folder. Each run picks up only the files that arrived since the last one. (This checkpoint is the "bookmark" in `04-orchestration-airflow.md`.)
2. **Clean each row.** Times become UTC, and status words are translated through a small mapping table.
3. **Attach the customer ID.** The job looks up each payment's account in the crosswalk table `customer_id_map` and adds the company-wide ID, e.g. `acct_77812` → `C-000981`.
4. **Keep the latest version of each payment.** If the batch holds several events for `T-1001`, the job keeps the one with the highest LSN (Log Sequence Number), the event's position in the database's change log.
5. **Merge into Silver.** `MERGE` inserts new payments and updates existing ones, but only when the incoming event is newer than the row already there. Re-running the same files changes nothing.

The vendor jobs follow the same steps, with one difference in step 4: they have no LSN, so "latest" means the record's last-modified time.

### The customer ID crosswalk

`customer_id_map` has one row per source ID, all pointing to one company-wide ID. The links come from three places:

- **Payments ↔ Salesforce:** when a merchant signs up, the sign-up flow creates the Salesforce record and stores the payments account ID on it.
- **Mambu ↔ Salesforce:** when a customer takes a loan, the loan application writes the Mambu ID onto the same Salesforce record.
- **Persona ↔ Salesforce:** every identity check is started with the Salesforce ID as its reference, and Persona sends that reference back with the result.

So for recent customers the job just reads the links. **Older customers** signed up before those fields existed. For them we ran a one-time match between Salesforce and Mambu on email and legal business name. Strong matches ("Jane's Bakery LLC" and "Janes Bakery, LLC" with the same email) were linked automatically. Uncertain ones, such as the same email with a different business name, went to a review list that an operations person worked through. A wrong link would mix two businesses' money, so we never guess.

### Two kinds of duplicates

Bronze keeps every copy. Silver ends up with one row per real thing.

| | Why it happens | How Silver spots it |
|---|---|---|
| **Live path** | After a restart, Debezium can send an event twice | Same table, same primary key, same LSN: the identical event, so the repeat is dropped. A *different* LSN for the same key is a real later change. |
| **Scheduled path** | Every pull lands in its own folder, and Persona checks arrive by webhook *and* in the daily pull | Same record ID and same last-modified time: identical, keep one. A later last-modified time is a newer version, so it replaces the old one. |

A delete in the payments database arrives as a delete event. Silver marks that row as deleted instead of removing it, so reports can still explain what happened.

### When each job runs

Airflow starts every job (`04`):

| Job | Runs | Why |
|---|---|---|
| Payments (`transactions`, `refunds`, `chargebacks`) | Every 15 minutes | Bronze fills all day from Kafka |
| Loans (Mambu) | Hourly, right after the pull | Matches the hourly pull |
| Ledger, settlements, customers, identity checks | Daily, right after each pull | Those sources arrive once a day |

The 15-minute job takes its machines from a **pool** that Databricks keeps warm, so it starts in well under a minute. Starting cold machines takes several minutes, which is fine for a daily job but not for one that runs 96 times a day.

### Keeping it fast

Any step that matches rows by a key needs rows with the same key on the same machine. Spark has two ways to get them there, and the job uses each where it fits:

- **Broadcast, for the customer ID lookup.** `customer_id_map` is small (a few million short rows), so Spark copies the whole table to every machine. Each machine then matches its own share of payments locally, and the payments never move. On a 150-million-row re-run, that avoids sending all those rows across the network.
- **Shuffle, for keeping the latest version.** To find the latest `T-1001`, every copy of `T-1001` must be on one machine, so Spark moves rows between machines by key. Moving data over the network is the slow part of any Spark job, but a 15-minute batch is small, so it's cheap here.

## What broke

**The tips that never arrived.** After "Who is Blank?" (`02`), the Silver job read every column from Bronze. That still wasn't enough. When the company launched tipping at checkout, the payments team added a `tip_amount` column to the `transactions` table. As a new column it passed the schema registry and reached Bronze. But by default a Delta `MERGE` fills only the columns the Silver table already has and quietly drops the rest. So `silver.transactions` never got the tips, and every job stayed green. We found out when finance asked for tip totals and Silver had none.
- *Fix:* the merge now runs with schema evolution (`MERGE WITH SCHEMA EVOLUTION`), so a new column is added to the Silver table automatically, with nulls for older rows. After each run the job also compares the table's columns with the previous run's and alerts on any new one, so a person confirms it was meant.

## Sample data

**`customer_id_map`:**

| source | source_id | customer_id |
|---|---|---|
| salesforce | CUST-4821 | C-000981 |
| mambu | 12345 | C-000981 |
| payments | acct_77812 | C-000981 |

**The status mapping table:**

| source | source_status | status |
|---|---|---|
| payments | pending | pending |
| payments | disputed | disputed |
| adyen | Settled | settled |
| adyen | Chargeback | disputed |

**Jane's payment, Bronze to Silver.** Bronze holds every event; Silver holds the latest:

| Layer | txn_id | account / customer | status | lsn |
|---|---|---|---|---|
| Bronze | T-1001 | acct_77812 | pending | 500 |
| Bronze | T-1001 | acct_77812 | settled | 640 |
| Bronze (sent twice after a restart) | T-1001 | acct_77812 | settled | 640 |
| **Silver** | T-1001 | C-000981 | settled | 640 |

Compliance and security for this layer, including the masks on identity columns in `silver.customers`, are in `08-compliance.md`.
