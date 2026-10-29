# Rules for `augment/computer-vision-pytorch/`

Read `augment/RULES.md` first; this file adds the decisions specific to this augment.

## What it attaches to

`project-stories/fintech/`, at the loan application step: the application app creates the loan in Mambu (`03-processing-spark-databricks.md`). Everything is digital: no paper, scans, pens, or checks. Never edit `project-stories/` or the Cortex augment for this.

## Stage

Stage 1: one file only, `01-architecture-and-statement-tamper-detector.md`, with four sections: problem statement, pipeline (one diagram), explanation, user story.

## The one use case

- **Problem:** applicants who upload PDF bank statements instead of connecting through Plaid can edit them to look richer. This is a known, common fraud in small-business lending.
- **Solution:** file and arithmetic rules plus a PyTorch vision model that boxes edited areas, served live so the answer comes while the applicant is still in the app.
- **A flag never rejects:** it asks for a Plaid connection or sends the application to an underwriter. A person makes every decision.
- No second use case. Other CV (computer vision) or ML (machine learning) topics wait for stage 2's Q&A (question and answer) file.

## Fixed names (keep identical across files)

- Endpoint: `statement_check` (Databricks Model Serving, GPU (Graphics Processing Unit))
- Model: `statement_tamper_detector`, loaded by alias `@champion`
- Bronze: `s3://company-raw/loan-applications/statements/date=.../`
- Tables: `silver.statement_checks`, `gold.fct_statement_checks`
- Mambu status while checked: *Pending Approval*
- Examples: Joe's Grill (fraud, loan `L-9034`); Jane's Bakery (honest, flagged, cleared through Plaid)
