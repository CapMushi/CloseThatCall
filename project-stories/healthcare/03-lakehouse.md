# Lakehouse

## What it does

The lakehouse turns the raw data in Bronze into tables the bed board and reports can read. It has three stages, all Delta tables on Azure Data Lake Storage (ADLS Gen2), all processed by Azure Databricks:

| Stage | Table | One row is | Built from | Updated |
|---|---|---|---|---|
| **Bronze** | `bronze.adt_messages`, plus the nightly files | Exactly what arrived (`02-ingestion.md`) | The two EHRs (electronic health record systems) | Live, and nightly |
| **Silver** | `silver.adt_events` | One ADT (admit, discharge, transfer) message, parsed into columns | `bronze.adt_messages` | Live |
| | `silver.beds` | One bed | Epic `CLARITY_BED`, Oracle Health `Location` | Nightly |
| | `silver.patients` | One patient record from one EHR, with its network-wide ID | Epic `PATIENT`, Oracle Health `Patient` | Nightly |
| | `silver.stays` | One hospital stay, in the same shape for both EHRs | The other Clarity tables and FHIR resources | Nightly |
| **Gold** | `gold.bed_status` | One bed and who is in it right now | `silver.adt_events` + `silver.beds` | Live |
| | `gold.stay_kpis` | One stay with its boarding time and discharge wait | `silver.stays` | Nightly |
| | `gold.readmissions` | One return within 30 days | `silver.stays` + `silver.patients` | Nightly |

Silver is where the two EHRs stop looking different: one set of columns, one clock (UTC), and one ID per patient. Gold answers the three questions the network asked for: which beds are free, how long patients wait, and who came back.

## Why these tools (and what we didn't pick)

- **Delta tables.** Plain files can only be added to. Delta lets us update a row, which we need all the time: a stay changes for weeks after discharge, and a bed changes every time a patient moves.
- **Lakeflow Spark Declarative Pipelines** (formerly Delta Live Tables). We write each table as a query: "this table is that table, cleaned like so". Databricks works out the order, processes only new data, retries failures, and runs the same code as a live stream or a nightly batch. Data quality rules, called **expectations**, sit right on the table.
  - *Not hand-written Structured Streaming jobs:* we'd have to write and maintain the progress tracking, the ordering and the upserts ourselves. We use plain Structured Streaming only for the one simple job that lands messages in Bronze.
  - *Not dbt:* it's built for SQL batch models. Our Silver needs a live stream and Python code to parse HL7 v2, the hospital message format.
- **Splink** for patient matching. It's an open-source library for probabilistic record linkage: it scores how likely two records are the same person, and it runs on the Spark we already have.
  - *Not exact matching:* it misses anyone whose name is typed differently, like Margaret, who is "Peggy" at St. Anne's.
  - *Not a commercial master patient index:* those products exist to merge charts inside the EHRs, with clinical sign-off. We only need to link records for reporting, and we never write anything back to an EHR.

## How it works

### Where the tables live

Storage account `stnetworklake` has one container per stage, and Unity Catalog maps each one to a schema in the `network` catalog:

```
abfss://bronze@stnetworklake.dfs.core.windows.net/   ← network.bronze   (layout in 02-ingestion.md)
abfss://silver@stnetworklake.dfs.core.windows.net/   ← network.silver   adt_events, beds, patients, stays
abfss://gold@stnetworklake.dfs.core.windows.net/     ← network.gold     bed_status, stay_kpis, readmissions
```

Silver and Gold tables are created and managed by the pipelines, so people and reports reach them by name (`network.gold.bed_status`), never by folder path.

### Two pipelines

| Pipeline | Runs | Builds |
|---|---|---|
| `adt_live` | Continuously | `silver.adt_events` → `gold.bed_status` |
| `nightly_build` | Once a night, after both extracts land (`05-operations.md`) | `silver.beds`, `silver.patients`, `silver.stays` → patient matching → `gold.stay_kpis`, `gold.readmissions` |

### Live: from a message to a bed

Margaret is admitted to bed 4W-12 at Northside (`02`). Her message is in Bronze as one long string.

1. **Parse.** A small, tested Python function splits the message into segments and fields and picks out what we need: the event type (`A01`), when it happened, her MRN (medical record number), and the unit and bed. Local hospital times become UTC.
2. **Check.** Expectations decide what happens to a bad row:

   ```python
   from pyspark import pipelines as dp

   @dp.table(name="silver.adt_events")
   @dp.expect_or_drop("has_message_id", "message_control_id IS NOT NULL")
   @dp.expect_or_drop("has_mrn", "mrn IS NOT NULL")
   @dp.expect_or_drop("known_event", "event_type IN ('A01','A02','A03','A04','A08','A11','A12','A13')")
   @dp.expect("not_in_future", "event_time <= current_timestamp()")
   def adt_events():
       return parse_hl7(spark.readStream.table("network.bronze.adt_messages"))
   ```

   | Rule type | A failing row is | We use it for |
   |---|---|---|
   | `expect` | Kept, and counted | Things worth watching, like an event time in the future |
   | `expect_or_drop` | Left out of Silver, and counted | Rows nothing can be built on, like a message with no MRN |
   | `expect_or_fail` | Stops the run | Nightly tables only, e.g. a stay with no patient. A broken extract must not reach a report. |

   The live pipeline never uses `expect_or_fail`: one bad message shouldn't stop the bed board. A dropped row is never lost, because the raw message is still in Bronze.
3. **Find where each patient is now.** The pipeline keeps only each patient's latest event, ordered by when it *happened*, not when it arrived. Lakeflow's **AUTO CDC** feature (CDC is change data capture) does this: we tell it the key (EHR + MRN) and the ordering column (event time), and it handles messages that arrive late or twice. An admit or transfer puts the patient in a bed; a discharge takes them out.
4. **Join to the bed list.** `silver.beds` has every bed in the network. `gold.bed_status` joins the two: each bed with a patient in it shows as `occupied`, and every other bed as `not occupied`. Within minutes of the message, the board shows 4W-12 as taken.

What happens when messages arrive out of order, twice, or get cancelled is its own story, in `07-problems-solved.md`.

### Nightly: one shape for two EHRs

Epic's data arrives as Clarity tables and St. Anne's as FHIR (Fast Healthcare Interoperability Resources) records (`02`). Silver maps both into the same columns, so nothing after Silver needs to know which EHR a row came from:

| `silver.stays` column | Epic source | St. Anne's source |
|---|---|---|
| Stay ID, admitted, discharged | `PAT_ENC_HSP` (the stay ID is its `PAT_ENC_CSN_ID`) | `Encounter` id and period |
| Arrived in ED (emergency department), left ED | `CLARITY_ADT` | `Encounter` location history |
| Admit decision, discharge order | `ORDER_PROC` | `ServiceRequest` |
| Diagnosis | `HSP_ACCT_DX_LIST` | `Condition` |
| Medicare or not | `HSP_ACCOUNT` | `Coverage` |

The same stay arrives many nights in a row, because we re-pull the last 30 days and coders keep finalising diagnoses. So `silver.stays` is also built with AUTO CDC: the key is `stay_id` (the EHR's own ID for the stay, with the EHR in front, e.g. `oracle-97553081`), the ordering column is the load date, and the newest version of each stay replaces the older one.

### Patient matching

This step runs between Silver and Gold. It compares every Epic patient record with every St. Anne's record that could plausibly be the same person, and gives matched records one network-wide ID.

1. **Standardise.** Names in upper case, "Street" and "St" made the same, phone numbers as digits only.
2. **Block.** Comparing every record with every other would be billions of pairs. Splink only compares pairs that share something, such as the same date of birth or the same phone number.
3. **Score each pair.** Splink compares the pair field by field and adds up the evidence into one match score:

   | Field | Epic | St. Anne's | Evidence |
   |---|---|---|---|
   | First name | Margaret | Peggy | Against |
   | Last name | Lopez | Lopez | For (a rarer surname would count for more) |
   | Date of birth | 1947-03-02 | 1947-03-02 | Strongly for |
   | Sex | F | F | Weakly for |
   | Address | 14 Elm St | 14 Elm Street | For, after standardising |
   | Phone | 555-0142 | 555-0142 | Strongly for |

   How much each field counts isn't typed in by us. Splink learns it from the data: how often a field agrees among real matches, and how often it agrees by pure chance.
4. **Decide.** There are two cut-offs. Above the higher one, the records are linked automatically. Between the two, the pair goes to a review list for the health information management team, who already resolve duplicate records in the EHRs. Below the lower one, nothing is linked.
5. **Assign the ID.** Linked records share one `network_patient_id`, and it never changes once given. Margaret's two records both get `P-000417`.

How we tuned this, and what went wrong first, is in `07`.

### Gold: the KPIs and readmissions

- **`gold.stay_kpis`** works out two waits for every stay: ED boarding time (admit decision until the patient leaves the ED) and discharge order to departure. `04-serving.md` summarises them by hospital and unit.
- **`gold.readmissions`** takes every discharge and looks for the same `network_patient_id` being admitted at any of our four hospitals within 30 days. It flags the ones that fall under Medicare's program: a Medicare patient whose first stay was for one of the six conditions.

This is an early warning, not Medicare's own calculation. Medicare also counts returns to hospitals outside our network and adjusts for how sick each hospital's patients are, and we can't see either.

## Sample data

**`silver.adt_events`**: Margaret's admit message, parsed (times in UTC; 10:12 at Northside is 16:12 UTC):

| hospital | message_control_id | event_type | event_time | source | mrn | patient_class | unit | bed |
|---|---|---|---|---|---|---|---|---|
| NORTHSIDE | MSG000183422 | A01 | 2026-03-01 16:12 | epic | E1234567 | I | 4W | 4W-12 |

**`gold.bed_status`**, a few minutes later:

| hospital | unit | bed | status | mrn |
|---|---|---|---|---|
| Northside | 4 West | 4W-12 | occupied | E1234567 |
| Northside | 4 West | 4W-13 | not occupied | |

**`silver.stays`**: her two stays, in one shape (times in UTC):

| network_patient_id | stay_id | hospital | admitted | discharged | arrived_ed | admit_decision | left_ed | discharge_order | diagnosis | medicare |
|---|---|---|---|---|---|---|---|---|---|---|
| P-000417 | epic-7004418823 | Northside | 2026-03-01 16:12 | 2026-03-06 21:40 | | | | 2026-03-06 17:05 | heart failure | yes |
| P-000417 | oracle-97553081 | St. Anne's | 2026-03-18 19:05 | | 2026-03-18 19:05 | 2026-03-18 20:40 | 2026-03-18 23:20 | | | yes |

**`gold.stay_kpis`**:

| network_patient_id | stay_id | hospital | admitted | ed_boarding_minutes | discharge_wait_minutes |
|---|---|---|---|---|---|
| P-000417 | epic-7004418823 | Northside | 2026-03-01 | | 275 |
| P-000417 | oracle-97553081 | St. Anne's | 2026-03-18 | 160 | |

A blank means there's nothing to measure: she went straight to a bed at Northside without passing through the ED, and she hasn't left St. Anne's.

**`gold.readmissions`**:

| network_patient_id | first_stay | discharged | came_back_to | readmitted | days_between | first_stay_reason | counts_for_medicare_program |
|---|---|---|---|---|---|---|---|
| P-000417 | Northside | 2026-03-06 | St. Anne's | 2026-03-18 | 12 | heart failure | yes |

How the PHI (protected health information) in these tables is secured is in `06-compliance.md`.
