# Rules for `augment/snowflake-cortex-genai/`

Read `augment/RULES.md` first; this file adds the decisions specific to this augment.

## What it attaches to

`project-stories/fintech/`. Cortex runs inside Snowflake, so everything here builds on the Snowflake side of that pipeline (Gold), plus one new document source described below.

## Files

Story files, in reading order:
1. `01-architecture-and-cortex-analyst.md`: the architecture (attachment points, layer placement, one diagram, front-end history), then use case 1.
2. `02-loan-documents-ai-functions-rag.md`: use case 2.
3. `03-cortex-agents-cowork.md`: one Cortex Agent that orchestrates use cases 1 and 2, and the front-end migration to CoWork.

Plus one Q&A file: `04-cortex-interview-qa.md`.

## Use cases: exactly two

- **Ask Finance (Cortex Analyst on Gold).** Fixes problem #1 in `00-problem-statement.md`: three different answers to "how much did we make." A semantic view encodes Gold's certified metric definitions, so plain-English questions get the same numbers as the certified dashboards.
- **Loan-agreement documents (Cortex AI Functions + RAG (Retrieval-Augmented Generation)).** Signed loan PDFs are parsed and their key terms extracted, then checked against Mambu with a dbt test. Cortex Search over the agreement text lets a reviewer read what the contract says when a mismatch is flagged.
- No third use case (no AML investigation agent, no fraud). Anything else the JD asks about goes in the Q&A file.

## Cortex Agents

Cortex Agents appear only as the orchestrator of the two use cases. The agent's tools are the semantic view from use case 1 and the Cortex Search service from use case 2. Its reason to exist is questions that need both: numbers from Gold *and* what the contract says. A missing privilege on any tool fails the whole agent request, so users without contract access get a second, Analyst-only agent (use case 1 alone), never a new use case.

## Front end

- **Today (before):** users worked in a Streamlit in Snowflake app with two tabs, one per use case. They had to know which tab to use, and a question needing both had to be split by hand.
- **Migration:** moving to Snowflake CoWork (renamed from Snowflake Intelligence in 2026) because it gives one chat entry point where the agent routes between tools, built-in Cortex AI Guardrails, agent monitoring, and sharing agents by Snowflake role.
- Tell it as a migration in progress, with phases and what remains (confirm).

## Layer placement

- **Bronze:** signed PDFs land untouched in S3, like every other source.
- **Snowflake reads them directly from Bronze, skipping Databricks.** Cortex AI Functions only run in Snowflake, so routing PDFs through Spark adds a hop and cost for no work. State this exception and its reason explicitly.
- **Silver equivalent:** the `docs` schema in Snowflake (parsed text, chunks, extracted terms).
- **Gold:** the mismatch model only.
- **Serving:** the Cortex Search service and the agent.
- RAG never sits in Gold. Gold is for certified business metrics.

## Fixed names (keep identical across files)

- Bronze path: `s3://company-raw/docusign/date=.../`
- Semantic view: `finance_semantic_view`
- Document tables: `docs.loan_agreement_text`, `docs.loan_agreement_chunks`, `docs.loan_agreement_terms`
- Mismatch model: `gold.loan_terms_mismatch`, comparing against `gold.dim_loan` (Mambu's loan terms, one row per loan)
- Search service: `loan_agreement_search`
- Agents: `finance_lending_assistant` (both tools), `finance_metrics_assistant` (Analyst only)
