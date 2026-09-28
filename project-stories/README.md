# Project Stories

One folder per project you can talk about in interviews, named `<domain>-<short-name>/` (e.g. `fintech-fraud-detection/`, `healthcare-claims-pipeline/`).

Each story is split across several files by major pipeline tech/stage rather than one giant file — easier to pull just the relevant chunk when a company cares about one part of the stack (e.g. only the Snowflake piece, or only orchestration). Copy `_template/` to start a new one and rename/drop files to match the actual pipeline.

Suggested file breakdown (adjust stages to match the real pipeline):

- `00-problem-statement.md` — the business problem, in plain terms, with your role, team size, and timeline
- `01-pipeline-overview.md` — one diagram of the whole pipeline start to end (main building blocks only), so a reader gets the overall shape before the detail
- `02-ingestion-<tech>.md` — e.g. `02-ingestion-kafka.md`
- `03-processing-<tech>.md` — e.g. `03-processing-spark.md`
- `04-orchestration-<tech>.md` — e.g. `04-orchestration-airflow.md`
- `05-storage-warehouse-<tech>.md` — e.g. `05-storage-warehouse-snowflake.md`
- `06-serving-analytics-<tech>.md`
- `07-monitoring-observability-<tech>.md`
- `08-challenges-tradeoffs.md` — decisions made, alternatives considered, what you'd do differently

Diagrams use Mermaid (fenced ` ```mermaid ` code blocks) directly inside the markdown file — no separate diagram files. VS Code renders them with the Markdown Preview Mermaid Support extension; GitHub renders them natively.

Link back to relevant `domains/` and `tech-stack/` notes instead of duplicating that content here.
