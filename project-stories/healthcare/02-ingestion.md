# Ingestion

## What it does

Ingestion gets data out of the two EHRs (electronic health record systems) and into Bronze, the raw layer of the lakehouse, without changing it. It runs on two paths:

| | Live path | Nightly path |
|---|---|---|
| **Carries** | ADT messages: admits, transfers, discharges | Full records: stays, orders, diagnoses, payer, patient details |
| **Tools** | Mirth Connect → Azure Event Hubs → Databricks streaming | Azure Data Factory |
| **Speed** | Seconds into Bronze | Once a night |
| **Feeds** | The bed board | KPIs, readmissions, patient matching |

Why both? ADT messages are fast, but they only say *where* a patient is. They don't carry orders, final diagnoses or who's paying, and they can be corrected later. The nightly records are complete and final, but a day old. The bed board needs speed; readmissions need completeness.

## Why these tools (and what we didn't pick)

**Live path**
- **Mirth Connect.** EHRs send HL7 v2 messages (the standard hospital message format) over MLLP (Minimal Lower Layer Protocol), a plain TCP protocol meant for the hospital's internal network. Something inside the network has to receive them and forward them to the cloud over HTTPS. The network already ran Mirth for its lab and radiology interfaces, so we added two channels to it instead of buying a second engine.
- **Azure Event Hubs.** A managed stream that keeps every message for 7 days, so we can replay if something downstream breaks. It also has a Kafka-compatible endpoint, so Databricks reads it with its standard Kafka reader.
  - *Not Azure Service Bus:* it's a queue, and a message is gone once it's read. We wanted replay.
  - *Not our own Kafka cluster:* more servers to run, for no gain at our size.
- *Not Azure Health Data Services (converting HL7 to FHIR on arrival):* it puts a database and a conversion step in front of the raw message. We wanted the raw message stored first, untouched.

**Nightly path**
- **Azure Data Factory.** Epic's Clarity database is a SQL Server inside the hospital data center. Data Factory's **self-hosted integration runtime**, a small agent installed on a server inside the network, reads Clarity and pushes the files out, so the data center never has to accept connections from the cloud. Data Factory also has the built-in steps for Oracle's start-wait-download export.
- **Bulk FHIR for St. Anne's.** FHIR (Fast Healthcare Interoperability Resources) is the newer, API-based health data standard. Every certified US EHR must offer a standard bulk FHIR export, so it's documented and supported, and we didn't need a custom extract from Oracle's database.
- *Not through Event Hubs:* it's built for a stream of small messages (at most 1 MB each) and keeps them for only 7 days. The nightly data arrives as large files that have to stay in Bronze anyway, so Data Factory copies them straight there.

## How it works

### Live path: Margaret is admitted at Northside

1. **Epic sends the message.** Margaret is admitted to bed 4W-12. Epic sends an `ADT^A01` (admit) message over MLLP to Mirth's Epic channel. One Epic feed covers all three Epic hospitals; St. Anne's has its own feed into a second channel.
2. **Mirth hands it on.** Mirth acknowledges the message back to Epic and posts it to the `adt-messages` event hub in the `evh-network-adt` namespace.
   - It uses the hospital (`NORTHSIDE`) as the **partition key**, so each hospital's messages stay in order within one partition.
   - If Event Hubs can't be reached, Mirth queues the message and keeps retrying, so nothing is lost.
3. **Databricks stores it in Bronze.** A Structured Streaming job reads the event hub through its Kafka endpoint and appends each message, unparsed, to the Delta table `network.bronze.adt_messages`. Silver and Gold pick it up from there (`03-lakehouse.md`), and the bed board shows her bed as taken within minutes.

**The messages we take:**

| Event | Meaning |
|---|---|
| A01 | Admit to a bed |
| A02 | Transfer (moved to another bed or unit) |
| A03 | Discharge |
| A04 | Register (e.g. a patient arriving in the emergency department, ED) |
| A08 | Update patient details |
| A11, A12, A13 | Cancel an admit, a transfer, or a discharge |

Exactly which events each EHR sends is part of its interface setup, so we agreed this list with each hospital's interface team.

### Nightly path

Two Data Factory pipelines write their files into Bronze. Once both have landed, Data Factory starts the nightly Lakeflow Jobs build (`05-operations.md`).

**Epic (`pl_epic_clarity_nightly`).** Epic rebuilds Clarity from its live database overnight. Once that load has finished, Data Factory copies seven tables to Bronze as Parquet files:

| What we need | Epic: Clarity table | St. Anne's: FHIR resource |
|---|---|---|
| Who the patient is (for matching) | `PATIENT` | `Patient` |
| Each hospital stay | `PAT_ENC_HSP` | `Encounter` |
| Bed moves, including leaving the ED | `CLARITY_ADT` | `Encounter` location history |
| The list of beds | `CLARITY_BED` | `Location` |
| Admit-decision and discharge orders | `ORDER_PROC` | `ServiceRequest` |
| Final coded diagnoses | `HSP_ACCT_DX_LIST` | `Condition` |
| Payer (Medicare or not) | `HSP_ACCOUNT` | `Coverage` |

Each night we pull every stay that was open or discharged in the last 30 days, not just yesterday's. Medical coders finalise a stay's diagnoses days after discharge, so last week's stays keep changing.

**St. Anne's (`pl_oracle_fhir_nightly`).** Bulk FHIR is asynchronous: you ask, wait, then download.
1. **Sign in.** Data Factory gets an access token for our Oracle system account, using SMART Backend Services, the standard machine-to-machine login for bulk FHIR: we sign a short request with a private key kept in Azure Key Vault and get a token back.
2. **Ask.** It calls `GET [base]/Group/[group-id]/$export` for St. Anne's patient group, with `_type=Patient,Encounter,Location,ServiceRequest,Condition,Coverage` and `_since` set to the start of the last successful run, so only changed records come back.
3. **Wait.** Oracle replies "accepted" with a status URL. Data Factory checks it, backing off between checks, until the export is done.
4. **Download.** The final status lists the NDJSON files (one JSON record per line, one resource type per file). Data Factory copies each one to Bronze and saves the new `_since` time.

Two Oracle rules shape this: only one export can run at a time for our account, and the files expire after 30 days, so we download them the same night.

### Where everything lands

Storage account `stnetworklake`, container `bronze`:

```
abfss://bronze@stnetworklake.dfs.core.windows.net/
  adt_messages/                                  ← Delta table network.bronze.adt_messages (live)
  _checkpoints/adt_messages/                     ← streaming job's progress marker
  epic_clarity/PATIENT/load_date=2026-03-07/part-0000.parquet
  epic_clarity/PAT_ENC_HSP/load_date=2026-03-07/...
  epic_clarity/CLARITY_ADT/load_date=2026-03-07/...
  epic_clarity/CLARITY_BED/load_date=2026-03-07/...
  epic_clarity/ORDER_PROC/load_date=2026-03-07/...
  epic_clarity/HSP_ACCT_DX_LIST/load_date=2026-03-07/...
  epic_clarity/HSP_ACCOUNT/load_date=2026-03-07/...
  oracle_fhir/Patient/load_date=2026-03-19/Patient_1.ndjson
  oracle_fhir/Encounter/load_date=2026-03-19/Encounter_1.ndjson
  oracle_fhir/Location/load_date=2026-03-19/...
  oracle_fhir/ServiceRequest/load_date=2026-03-19/...
  oracle_fhir/Condition/load_date=2026-03-19/...
  oracle_fhir/Coverage/load_date=2026-03-19/...
```

Nothing in Bronze is ever edited. Each night adds a new `load_date` folder. How Silver uses these files is in `03-lakehouse.md`; how the PHI (protected health information) in them is secured is in `06-compliance.md`.

## Sample data

**The admit message Epic sends** (HL7 v2: one segment per line, fields split by `|`):

```
MSH|^~\&|EPIC|NORTHSIDE|MIRTH|NETWORK|20260301101500||ADT^A01|MSG000183422|P|2.5.1
EVN|A01|20260301101500||||20260301101200
PID|1||E1234567^^^EPIC^MR||LOPEZ^MARGARET||19470302|F|||14 ELM ST^^SPRINGFIELD^IL^62701||555-0142
PV1|1|I|4W^12^^NORTHSIDE
```

- `MSH`: who sent it, the type (`ADT^A01`), and a unique ID (`MSG000183422`).
- `EVN`: when it was recorded (10:15) and when it really happened (10:12).
- `PID`: the patient: MRN (medical record number), name, date of birth, sex, address, phone.
- `PV1`: the stay: patient class (`I` = inpatient) and location (unit `4W`, room `12`, Northside).

**The same message in Bronze**, stored whole, plus where it came from:

| hospital | message_type | message_control_id | enqueued_time | partition | offset | raw_message |
|---|---|---|---|---|---|---|
| NORTHSIDE | ADT^A01 | MSG000183422 | 2026-03-01 10:15:02 | 2 | 418822 | `MSH\|^~\&\|EPIC\|NORTHSIDE\|…` |

**Margaret's St. Anne's stay from Oracle's bulk export** (one line in `Encounter_1.ndjson`, spread out here to read):

```json
{"resourceType": "Encounter", "id": "97553081", "status": "in-progress",
 "class": {"code": "IMP"},
 "subject": {"reference": "Patient/12724066"},
 "period": {"start": "2026-03-18T19:05:00Z"},
 "location": [
   {"location": {"reference": "Location/32216049", "display": "ED Bay 04"},
    "period": {"start": "2026-03-18T19:05:00Z", "end": "2026-03-18T23:20:00Z"}},
   {"location": {"reference": "Location/32217712", "display": "3E-07"},
    "period": {"start": "2026-03-18T23:20:00Z"}}]}
```

The location history shows her in the ED until 23:20, then in bed 3E-07, which is exactly what the boarding KPI needs. Note that `Patient/12724066` is Oracle's internal ID, not her MRN: the MRN (`S889201`) sits inside the `Patient` record, and that's what matching uses.
