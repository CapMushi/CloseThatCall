# Compliance and Security

The layer files explain how data moves. This file explains what keeps it safe and auditable. Each control appears here once, even when it touches several layers.

## What applies to us

| Rule | What it protects | What it means for the platform |
|---|---|---|
| **PCI-DSS** (Payment Card Industry Data Security Standard) | Card numbers | Keep full card numbers out of the platform entirely |
| **SOX** (Sarbanes-Oxley Act) | Financial reporting | Prove where every number came from, that nobody changed it unseen, and that the checks ran |
| **KYC/AML** (Know Your Customer / Anti-Money Laundering) | Customer identities; spotting suspicious money movement | Compliance needs Persona's identity data; everyone else must never see it |

## PCI-DSS: card numbers never enter the platform

- Adyen captures the card, not us. Our payments app and Adyen's files only ever hold Adyen's token for the card and its last 4 digits, never the full number, the PAN (Primary Account Number).
- So nothing in the platform (Kafka, S3, Databricks, Snowflake, Tableau) holds a card number. That keeps all of it out of the card-data audit.
- This only works because we can't turn a token back into a card number. Adyen holds that mapping, so Adyen stays in scope and we don't.
- Tableau goes one step further: its data sources leave out even the tokens and last 4 digits.

## KYC/AML: identity data only compliance can see

**Why it's in the platform at all:** AML monitoring means linking money that moved to the verified person behind it, so Persona's results are joined onto customers in Silver.

**The risk is the join itself.** An analyst builds "revenue by customer" with `SELECT *` over `silver.transactions` joined to `silver.customers`, and suddenly `id_document_number` sits in a table finance and product can read.

**So the identity columns are locked down at every stop:**

| Where | Control |
|---|---|
| Bronze | Persona's folder (`s3://company-raw/persona/`) is readable only by the KYC-compliance IAM (Identity and Access Management) role |
| Silver, in Databricks | Unity Catalog column masks on `id_document_number`, `sanctions_match` and `sanctions_list_details` in `silver.customers`. Anyone without the KYC-compliance role sees `***MASKED***`, even through a join, so the analyst's table only ever gets masked values. `kyc_status` (passed or failed) stays visible, because operations needs it. |
| Silver, read by Snowflake | Unity Catalog masks don't apply when Snowflake reads the files directly, so Snowflake's `silver` schema is readable only by the dbt service account |
| Gold | Snowflake masking policies on the same columns in `dim_customer` |
| Tableau | Identity columns are left out of every data source |

## SOX: prove every number

| What an auditor asks | How we answer |
|---|---|
| "Where did this number come from?" | Any Tableau figure drills down to its `fct_payments` rows. dbt records which table is built from which. Every NetSuite pull logs its source, row count, start and end time, and run ID. Every dashboard shows when its data is from. |
| "Could anyone have changed it?" | Bronze has S3 Object Lock in compliance mode: nobody, not even the AWS (Amazon Web Services) root account, can edit or delete a file before its retention date. Only the dbt service account writes Gold, and only from reviewed Git pull requests. Only data engineering can publish a certified Tableau data source or re-run a pipeline by hand (Airflow roles). |
| "What did it say back then?" | Recent past: Snowflake Time Travel (see below). Signed-off months: the month-end clone (see below). |
| "Did your checks actually run?" | Airflow's run history, stored dbt test results, and the alert history in Datadog and PagerDuty. Monitors are defined in Git through Terraform and changed only by review, so nobody can quietly switch an alert off. |

**Time Travel:** Snowflake keeps old versions of changed or dropped data for a set window (1 day by default, up to 90 days on Enterprise edition). Within it we can query a table as it was at 09:00 last Monday, undo a bad dbt deploy, or bring back a dropped table. After that, Fail-safe keeps 7 more days that only Snowflake support can restore.

**The month-end clone:**
- Finance signs off February in the first days of March, but February's data can still change afterwards (a late refund, a correction). Time Travel only reaches back 90 days, so it can't answer an auditor in July.
- So on close day we take a **zero-copy clone** of Gold, e.g. `gold_close_2026_02`: an instant copy that shares the stored data instead of duplicating it, so it costs almost nothing. Users get read access only.
- The clone answers "what did we sign off?" Live Gold answers "what's true now?" Finance books late changes as adjustments in the next month, and those explain the difference.

## Access and security across the platform

- **One login:** people sign in with the company's single sign-on, and their permissions come from their team group. Removing someone's login removes all their access.
- **Roles:** finance, risk and analysts each get a Snowflake role that reads only the Gold marts they need. Only service accounts can write.
- **Row-level security:** a small entitlements table in Gold says which user may see which customers, e.g. a relationship manager sees only their own portfolio. Tableau's certified data sources filter on it using the signed-in user's name.
- **Encryption and storage access:** every S3 bucket is encrypted with keys in AWS KMS (Key Management Service), and access is per role. For example, only the ingestion role can write to Bronze.
- **Secrets:** API (Application Programming Interface) keys and SFTP (Secure File Transfer Protocol) credentials live in AWS Secrets Manager and reach Airflow through its secrets backend. They're never in code, config files or Airflow's own database.
- **Data stays in our AWS account:** Databricks runs its machines inside our account, vendor pulls stream straight to S3 without copies on Airflow's workers, and payment data never passes through Airflow at all.
- **Logs hold counts and IDs, never records:** no card tokens or identity data reach Airflow or Datadog logs. Datadog's Sensitive Data Scanner is the backstop: it masks anything that looks like a card number before storing it. Logs are searchable for 15 days and archived to S3 for a year (confirm).
