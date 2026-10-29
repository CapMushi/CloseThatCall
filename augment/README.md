# Augment

Optional, JD (Job Description)-specific extensions that bolt onto an existing `project-stories/` pipeline — technology a particular interview emphasizes that isn't part of that project's core, deterministic story. Keeps the base story universal across interviews, while letting prep go deep on whatever stack one specific JD calls out, without rewriting or cluttering the base pipeline.

One subfolder per augment topic, named after the technology/theme, e.g.:

- `snowflake-cortex-genai/` — Snowflake Cortex, RAG (Retrieval-Augmented Generation), and Cortex Agents, bolted onto `project-stories/fintech/` (Gold layer plus a new loan-agreement document source)
- `computer-vision-pytorch/` — a PyTorch vision model that spots edited bank statements uploaded with loan applications, before the loan is approved in Mambu (stage 1: one file)

Read `RULES.md` before creating or editing any file here.
