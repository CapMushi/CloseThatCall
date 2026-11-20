# Rules for `project-stories/healthcare/`

Read before creating or editing any file here.

## The story

- One hospital-network data engineering story, told in first person by the senior data engineer who designed the platform. Standalone: no dates, no timeline.
- **The company:** a regional network of 4 hospitals that has grown by buying hospitals. 3 run Epic on one shared Epic system (one MRN per patient, shared bed view). 1, St. Anne's, was recently bought and still runs Oracle Health (formerly Cerner). Its move to Epic is planned for about 18 months out (typical for a hospital joining an existing Epic system).
- **The business problem** (problem statement only). Root cause: two EHRs.
  - **Bed capacity:** no single view of free beds across all 4 hospitals; St. Anne's beds exist only in Oracle Health.
  - **Readmissions:** a patient who moves between an Epic hospital and St. Anne's has a different MRN in each, so the network can't see the readmission. Medicare links stays through its own claims and the patient's Medicare ID, and counts it against the discharging hospital (6 conditions, any hospital, 30 days, risk-adjusted, cut of up to 3% on all its Medicare inpatient payments).
- **Why the platform outlasts the Epic move:** Medicare looks back 3 years; future acquisitions plug in the same way; every hospital gets the KPIs below.
- Nothing else: no claims files, no health information exchange, no outside hospitals (one line at most).

## KPIs (exactly three)

ED boarding time (admit decision → leaves the ED), discharge order → departure, 30-day readmission rate. One definition each, used at all four hospitals. Not included: occupancy (that's the bed board itself), bed turnaround and discharge-before-noon. Don't claim the ADT feeds carry bed cleaning status unless it's verified.

## Tech stack (fixed)

| Layer | Tool | Its job here |
|---|---|---|
| Sources | Epic (3 hospitals), Oracle Health (1 hospital) | The two EHRs |
| Live ingestion | Mirth Connect (paid license since 4.6) → Azure Event Hubs | Receives HL7 v2 ADT messages from both EHRs and posts each one to Event Hubs over HTTPS |
| Batch ingestion | Azure Data Factory | Nightly Epic Clarity extract; nightly Oracle Health bulk FHIR group export using `_since` |
| Storage | ADLS Gen2, Delta tables | Bronze, Silver, Gold |
| Processing | Azure Databricks: Structured Streaming, Lakeflow Spark Declarative Pipelines (formerly Delta Live Tables) with expectations, Splink | ADT stream, cleaning and quality rules, patient matching |
| Gold and SQL | Databricks SQL + Unity Catalog | No separate warehouse |
| Orchestration | Lakeflow Jobs (formerly Databricks Workflows), triggered by Data Factory | Runs the nightly build once extracts land |
| BI | Power BI | Bed board and readmissions reports |
| Monitoring | Azure Monitor + Databricks system tables | Pipeline and data health |
| Governance | Unity Catalog (row filters, column masks, lineage); Microsoft Purview | Purview appears only in `06-compliance.md` |

No augment tech in this story.

## Files

```
01-overview.md          problem, solution, one diagram of the whole pipeline, one patient's journey
02-ingestion.md         live ADT path and nightly batch path
03-lakehouse.md         storage layout, Bronze → Silver → Gold, patient matching
04-serving.md           Databricks SQL, Power BI, the bed board, KPI and readmission reports
05-operations.md        orchestration and monitoring
06-compliance.md        HIPAA, PHI, de-identification, access
07-problems-solved.md   the 3 problem stories
```

- Build one file at a time. After each, check it for gaps and mismatches against the earlier files before starting the next.
- **Layer files (02–05) use these sections only:** What it does → Why this tool (and what we didn't pick) → How it works, explained through what it does in this story → Sample data. Add a section only if it's truly needed.

## The 3 problem stories (all in `07`, nowhere else)

No per-file gotchas. Exactly three stories, each explainable in a sentence or two:
1. **Technical: one patient, two IDs.** Cross-EHR patient matching with Splink.
2. **Technical: a patient in two beds.** HL7 ADT messages arriving out of order, twice, or cancelled (A11, A13).
3. **Non-technical: "show me one."** The bed board counted any bed without a patient as available. Nursing, housekeeping and data engineering each held part of the truth. Brings in cross-team work, stakeholders and mentoring a junior engineer. Funny, never cheesy.

## Writing

- Simple, conversational English, like explaining to a colleague. Short sentences. Technical depth where it earns its place, nothing more.
- Tools and platform come first. Healthcare terms get a few plain words where they're first used.
- Expand healthcare and uncommon acronyms once per file. Skip ones every engineer knows (SQL, API, ETL).
- Numbers only where they're truly needed. No "(confirm)" tags.
- Realism first: research anything about product behaviour, regulation, or hospital operations before writing it. If it can't be verified, leave it out.
- Write every storage path and folder layout out in full. Show examples as small sample-data tables.
- No fluff: drop any detail that doesn't add real depth.
- When adding to an existing file, work the new content into what's already there: put it where it belongs, merge it with related lines, and remove anything it repeats. Never just drop a new line or block in.
- Mermaid diagrams in fenced blocks, only where they earn their place. Render and look at each one before calling a file done (scratchpad, `@mermaid-js/mermaid-cli`).
- Compliance lives only in `06`. Other files may point to it in one line.
- Never edit `project-stories/fintech/` or `augment/` for this story.

## Fixed names

Added here as each file introduces them, so later files stay consistent.

- **Hospitals:** Northside, Lakeview, Central (Epic); St. Anne's (Oracle Health, recently bought)
- **Example patient:** Margaret "Peggy" Lopez, born 1947-03-02, F, 14 Elm St, phone 555-0142, heart failure. Epic MRN `E1234567`, Oracle Health MRN `S889201`, network ID `P-000417`. Discharged from Northside bed `4W-12` on 2026-03-06; readmitted at St. Anne's on 2026-03-18 (12 days).
- **Network patient ID format:** `P-` plus six digits
- **Margaret's stays:** Northside admitted 2026-03-01 10:12 to `4W-12`, discharged 2026-03-06 15:40; St. Anne's ED arrival 2026-03-18 19:05Z, bed `3E-07` at 23:20Z, Oracle Encounter `97553081`, Oracle Patient ID `12724066`; Northside Epic stay CSN `7004418823`. Silver `stay_id` = EHR prefix + the EHR's stay ID (`epic-7004418823`, `oracle-97553081`)
- **Live path:** Event Hubs namespace `evh-network-adt`, event hub `adt-messages` (partition key = hospital code, 7-day retention); Bronze table `network.bronze.adt_messages`
- **Storage:** account `stnetworklake`, container `bronze`; folders `epic_clarity/<TABLE>/load_date=YYYY-MM-DD/`, `oracle_fhir/<Resource>/load_date=YYYY-MM-DD/`
- **Unity Catalog:** catalog `network`, schemas `bronze`, `silver`, `gold`
- **Data Factory pipelines:** `pl_epic_clarity_nightly`, `pl_oracle_fhir_nightly`
- **Clarity tables:** `PATIENT`, `PAT_ENC_HSP`, `CLARITY_ADT`, `CLARITY_BED`, `ORDER_PROC`, `HSP_ACCT_DX_LIST`, `HSP_ACCOUNT`. **FHIR resources:** `Patient`, `Encounter`, `Location`, `ServiceRequest`, `Condition`, `Coverage`
- **Storage containers:** `bronze`, `silver`, `gold` in `stnetworklake`, one per Unity Catalog schema
- **Silver tables:** `silver.adt_events`, `silver.beds`, `silver.patients`, `silver.stays`. **Gold tables:** `gold.bed_status` (status is `occupied` or `not occupied`), `gold.stay_kpis`, `gold.readmissions`
- **Lakeflow pipelines:** `adt_live` (continuous), `nightly_build` (Silver → patient matching → Gold)
- **Times:** Silver and Gold store UTC. Northside is UTC-6 in early March 2026, so 10:12 local is 16:12 UTC
- **Margaret's KPI times:** Northside discharge order 2026-03-06 11:05 local (17:05 UTC), departure 15:40 local, wait 275 minutes; St. Anne's admit decision 2026-03-18 20:40Z, left ED 23:20Z, boarding 160 minutes
- **Patient matching:** Epic records vs St. Anne's records; blocked on date of birth or phone; two cut-offs; middle band reviewed by the health information management team; never written back to an EHR
