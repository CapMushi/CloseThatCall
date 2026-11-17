# Rules for `augment/dfir-incident-response/`

Read `augment/RULES.md` first; this file adds the decisions specific to this augment.

## What it attaches to

`project-stories/fintech/`, at two existing points:
- **Snowflake Gold** (`05-storage-warehouse-s3-snowflake.md`): the system the attacker reached. Its masking policies and token-only card data limit what was exposed.
- **Datadog monitoring** (`07-monitoring-observability-datadog.md`): where the incident is first detected.

Never edit `project-stories/` or the other augments for this.

## Files (agreed with the user, replacing the stage-1 single file)

1. `01-problem-and-pipeline.md`: problem statement, what we must answer, and the overall 4-phase pipeline (one diagram).
2. `02-four-phases-and-glossary.md`: each phase in simple English (what happens, tools and platforms, worked example, output), then a glossary of the JD's skills, each with a plain meaning and an example from this case.

No fluff: only what's needed to explain and defend the investigation in an interview.

## The one incident

- An infostealer on one analyst's Windows laptop steals a password saved in the browser, including their Snowflake login. Four days later the attacker logs into Snowflake from an unknown IP and exports data.
- The account has a network policy (IP allowlist). The gap is a forgotten user-level "allow any IP" exception that overrides it, plus a password login through the Python connector before Snowflake's October 2026 MFA deadline.
- It's modelled on the real 2024 Snowflake customer breaches (stolen passwords, no MFA, no IP allowlist), kept small: one laptop, one user, one system reached.
- No second incident. JD topics the story doesn't use (Reverse Engineering, OT/ICS) are explained in the glossary as "not used here, and why".

## Fixed names (keep identical across files)

- Analyst laptop: `FIN-LT-0417` (Windows 11); Snowflake user: `SAM_RIVERA`
- Fake installer: `PDFConvert_Setup.exe`; attacker IP: `185.220.x.x` (shown truncated)
- Evidence bucket: `s3://company-forensics-evidence/case-2026-031/` (separate AWS account, Object Lock)
- Case timeline table: `case_2026_031.timeline` (Amazon Athena)
- Tools: CrowdStrike Falcon (EDR), Volatility 3, Autopsy and FTK Imager, Amazon Athena, Amazon Bedrock, Power BI
