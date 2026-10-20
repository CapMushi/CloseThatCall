# Interview Master

Personal knowledge base for interview prep (Data Engineer / AIML / Python roles). Not an application codebase — this is reference material, organized so content can be pulled together quickly per interview (a story, a domain briefing, a tech deep-dive, tracker history).

This structure is a first pass and open to change as real content gets added — reorganize freely if a folder stops making sense.

## Layout

- `interview-rounds/` — notes per round type (phone screen, technical, system design, behavioral, take-home)
- `sdlc/` — the SDLC process followed at past roles (agile flow, CI/CD, code review, testing, deployment)
- `project-stories/` — one folder per project, split across multiple files by major pipeline tech (see `_template/`)
- `first-round-story/` — one-file, plain-English version of the whole pipeline for a first-round interview, swapping in the JD's main technologies (currently Azure Data Explorer / KQL / SQL Server / Azure ML) and borrowing the rest from `project-stories/fintech/`
- `augment/` — optional, JD-specific tech extensions that bolt onto a `project-stories/` pipeline (e.g. a Snowflake Cortex/GenAI layer for an interview that emphasizes it) without rewriting the core story
- `domains/` — background per industry domain (fintech, healthcare, etc.) — vocabulary, regulations, common systems
- `tech-stack/` — reusable notes per technology (Snowflake, Azure, Spark, Airflow, ...), referenced by project stories
- `cv/` — CV versions (provided by you, not generated here)
- `tracker/` — `interview_tracker.xlsx`, log of interviews: company, domain, tech stack, round, outcome, why failed, notes
