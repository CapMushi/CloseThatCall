# Stolen Snowflake Login: The Four Phases and Glossary

**Attaches to:** `project-stories/fintech/`, Snowflake Gold (`05`) and Datadog monitoring (`07`). The incident and pipeline are in `01-problem-and-pipeline.md`.

DFIR (Digital Forensics and Incident Response) here means investigating the stolen Snowflake login (forensics) while stopping the attacker (incident response).

## Phase 1: Collect the evidence

**What happens:** before anything is cleaned up, copy every piece of evidence into one place nobody can alter, and record a fingerprint of each item.

| Evidence | From | How we collected it |
|---|---|---|
| Laptop RAM (memory) | `FIN-LT-0417` (Windows 11) | Falcon's network containment cut the laptop off from everything except Falcon itself. Falcon's remote shell (Real Time Response) then ran a memory-capture tool to save RAM to a file (confirm tool). |
| Laptop disk | `FIN-LT-0417` | Laptop couriered to IT; full disk copied with FTK Imager |
| Endpoint activity | CrowdStrike Falcon | Process, file and network events for the laptop, exported as JSON (JavaScript Object Notation) |
| Sign-ins | Microsoft Entra ID and on-premises Active Directory | Sam's sign-in logs, exported with PowerShell |
| Snowflake activity | `SNOWFLAKE.ACCOUNT_USAGE` views | `LOGIN_HISTORY`, `QUERY_HISTORY` and `ACCESS_HISTORY` exported with SQL. These views lag by up to about 3 hours, so live checks used `INFORMATION_SCHEMA` instead. |
| AWS activity | AWS CloudTrail | API (Application Programming Interface) calls for the last 30 days |

**Where it goes:** `s3://company-forensics-evidence/case-2026-031/`, in a separate AWS account that only the response team can reach. **S3 Object Lock** stops anyone, even an admin, from changing or deleting a file, the same feature that protects Bronze in the base story.

**Chain of custody:** every file gets a SHA-256 hash (a fingerprint that changes if a single byte changes), logged with who collected it, when and from where.

*Example:* `FIN-LT-0417_mem.raw`, 16 GB, SHA-256 `9f2c…e41a`, collected 7 Mar 04:05 UTC by on-call via Falcon.

**Output:** a locked, fingerprinted evidence set.

## Phase 2: Triage: separate the signal from the noise

**What happens:** about 2 million events were collected (confirm). Almost all are normal: email, Excel, Teams. Triage keeps only what's suspicious.

| Source | Tool | What we looked for | Found |
|---|---|---|---|
| RAM | **Volatility 3** | `windows.pslist`/`pstree`: running processes and their parents. `windows.cmdline`: the command each one ran. `windows.malfind`: code hidden inside another process. `windows.netscan`: network connections. | A process started by `PDFConvert_Setup.exe` had injected code into a normal-looking process, and had a connection to the attacker's server |
| Disk | **Autopsy** (FTK and X-Ways do the same job) | Download history, recently run programs, and anything set to restart at boot | The download from the ad at 14:20; Chrome's saved-password file opened by the malware at 14:21 |
| Snowflake logs | **SQL** | Logins from new IPs; queries by `SAM_RIVERA` outside working hours | One login from `185.220.x.x`; 14 queries; `ACCESS_HISTORY` shows exactly which columns were read |
| Whole company | **Falcon threat hunt** | The installer's hash on all ~1,200 laptops (confirm) | One other laptop, where Falcon had blocked it |
| Malware sample | **Malware triage** | Hash lookup on VirusTotal; run in a sandbox (a safe, isolated test machine) | A known infostealer family. No reverse engineering needed. |

*Example (Snowflake triage query):*
```sql
SELECT event_timestamp, user_name, client_ip, reported_client_type, is_success
FROM snowflake.account_usage.login_history
WHERE user_name = 'SAM_RIVERA'
  AND event_timestamp >= '2026-03-01'
ORDER BY event_timestamp;
```

**Output:** about 40 suspicious events (confirm), each pointing to the evidence file it came from.

## Phase 3: Analyse: connect the dots

**What happens:** put every suspicious event, from every source, on one timeline, then use AI to explain and group it faster. A person checks every claim.

- **One timeline:** a Python script turns each source's format into the same columns: `time_utc`, `source`, `host_or_user`, `event`, `evidence_ref`. It writes them as `case_2026_031.timeline`, which **Amazon Athena** queries with SQL straight from the evidence bucket. The evidence never leaves the forensics account.
- **AI-assisted analysis:** **Amazon Bedrock** runs a large language model inside our AWS account. Bedrock doesn't use prompts to train models, and model providers can't see them. We used it to:
  - explain encoded PowerShell commands and malware behaviour in plain English;
  - group the 14 queries by what they were after (customer list vs payments);
  - draft the timeline narrative for the report.
- **AI coding assistant:** GitHub Copilot drafted the parser for Falcon's JSON export. We reviewed and tested it against real events before using it.
- **AI governance (our rules, following the NIST AI Risk Management Framework):**
  1. Only approved, in-account AI tools (Bedrock); never public chatbots.
  2. Every AI statement must cite an `evidence_ref`. A person checks it against the evidence before it enters the report.
  3. Prompts and answers are logged with the case.
  4. AI never decides containment or notification. People do.

*Example:* the AI drafted "the attacker exported customer contact data". Checking `ACCESS_HISTORY` showed the identity columns were masked, so the reviewer corrected it to "business names and contact emails; identity documents masked".

**Output:** a verified timeline and a draft narrative.

## Phase 4: Report and fix

**What happens:** turn the timeline into something each audience can act on.

- **Power BI dashboard** (reads `case_2026_031.timeline` through Athena):
  - a timeline from the 4 Mar download to the 7 Mar lockout;
  - a scope panel showing the hosts, users and tables affected;
  - a status list for each fix.
- **One-page executive summary:** what happened, what was exposed, what was not, what we fixed, what we must notify.
- **Technical report:** each finding with its evidence reference and hash, for auditors and legal.
- **Fixes:**

| Gap | Fix |
|---|---|
| Password-only login through the Python connector | SSO (Single Sign-On) through Entra ID with MFA for every human user (Snowflake authentication policy) |
| User-level "allow any IP" exception overriding the account allowlist | Exception removed; a monthly check lists any user whose network policy differs from the account's; exceptions now expire |
| Passwords saved in the browser | Blocked by device policy; a password manager instead |
| Medium EDR alert ignored for 3 days | Infostealer alerts now page on-call |
| One stolen password reaches all Gold data | Analyst role reduced to the marts it needs |

**Output:** decisions made and gaps closed, with evidence to show auditors.

## Glossary

### Core skills (overlap between the CV and the JD)

| Term | What it means, simply | In this case |
|---|---|---|
| **Digital Forensics** | Collecting and examining digital evidence carefully enough that it would hold up in court | Hashing and locking the RAM and disk images |
| **Incident Response** | Stopping an active attack and recovering: contain, remove, fix | Disabling `SAM_RIVERA` and isolating the laptop |
| **Threat Hunting** | Searching proactively for signs of an attacker that no alert has flagged | Searching all ~1,200 laptops for the installer's hash |
| **Malware Triage** | Quickly working out what a malicious file is and does, without studying its code in depth | Hash lookup plus a sandbox run → "known infostealer" |
| **Memory Forensics** | Examining a copy of RAM to find programs and connections that may not exist on disk | Volatility found code injected into a normal-looking process |
| **Log Analysis** | Reading system records to reconstruct who did what, and when | Snowflake login and query history |
| **Attack Timeline Development** | Ordering every confirmed event from first entry to the end | 4 Mar download → 7 Mar export → 03:52 lockout |
| **Volatility** | Free, open-source tool for analysing RAM captures | `windows.malfind` revealed the injected code |
| **FTK** (Forensic Toolkit) | Commercial forensic suite; FTK Imager is its free disk-copying tool | Imaged the laptop's disk |
| **EnCase** | Commercial forensic suite, long used by law enforcement | Could replace Autopsy for the disk analysis |
| **X-Ways** | Fast, lightweight commercial forensic tool | Same disk analysis, quicker on big drives |
| **Autopsy** | Free, open-source forensic tool with a visual interface | Found the download and the password-file access |
| **Cloud Forensics** | Investigating cloud services, where there's no physical disk, only provider logs | Snowflake history and CloudTrail |
| **Cloud-based Investigations** | Running the investigation itself in the cloud: storage, queries, analysis | Evidence in S3, analysed with Athena and Bedrock |
| **Python** | Programming language for automating analysis | Turned five log formats into one timeline |
| **PowerShell** | Windows' command-line and scripting language, used by admins and attackers alike | Exported AD sign-ins; decoded the malware's commands |
| **SQL** (Structured Query Language) | The language for querying tables | Snowflake log queries; Athena timeline queries |
| **Power BI** | Microsoft's dashboard and reporting tool | The incident timeline dashboard |
| **Windows / Linux / macOS** | The three main operating systems; each stores evidence in different places | The laptop ran Windows 11; the same steps differ on Linux servers or Macs |
| **Active Directory** (AD) | Microsoft's directory of company users, computers and permissions | Showed Sam's normal sign-in pattern |
| **AWS** | Amazon's cloud, where our pipeline runs | Evidence account, CloudTrail, Athena, Bedrock |
| **Azure** | Microsoft's cloud; Entra ID is its identity service | Entra ID sign-in logs for Microsoft 365 |
| **GCP** (Google Cloud Platform) | Google's cloud | Not used here; its Cloud Audit Logs play the role of CloudTrail |
| **EDR Platforms** | Security agents on every laptop and server that record activity, raise alerts and can isolate a machine | CrowdStrike Falcon: alert, isolation, memory capture, fleet hunt |
| **Generative AI Tools** | AI that writes text or code from a prompt | Bedrock explained commands and drafted the narrative |
| **AI Coding Assistants** | AI built into the code editor that suggests code | Copilot drafted the JSON parser; we reviewed it |
| **AI Governance Frameworks** | Rules for using AI safely: what data, which tools, who checks | NIST AI RMF; our 4 rules in Phase 3 |

### Missing skills (asked for in the JD, not used in this case)

| Term | What it means, simply | Why not here, and how I'd approach it |
|---|---|---|
| **Reverse Engineering** | Taking a program apart to read how it works, using a disassembler such as Ghidra or IDA, often without its source code | The infostealer was a known family, so triage was enough. A new, unknown sample would go to a specialist (or the EDR vendor) to extract where it sends data and what it steals. |
| **OT/ICS** (Operational Technology / Industrial Control Systems) | Computers that run physical equipment: factories, power, pipelines | A fintech has none. The key difference: in OT, keeping the machine running comes first, so evidence is gathered passively (copies of network traffic) rather than by installing agents or rebooting. |
